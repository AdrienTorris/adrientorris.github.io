---
title: "Maestro: orchestrating an escape room, one event at a time"
date: 2026-11-12
lang: en
tags: [maestro, orchestration, mqtt, dotnet, event-driven, escape-room, architecture]
---

> **Alternative titles** (pick one, then delete this block)
>
> 1. The conductor plays no instrument: inside the Maestro engine
> 2. Notifications, milestones and delays: how Maestro plays an escape room
> 3. Writing a room's score: gameflow, concurrency and time in Maestro

# Maestro: orchestrating an escape room, one event at a time

An escape room looks a lot like a musical score. Over 60 minutes, hundreds of events happen: a player puts a statuette on a pedestal, types a code, turns a crank; a video ends; the game master (GM) clicks a button; time passes. Each one triggers reactions: a door unlocks, a light changes colour, music starts, smoke pours out, a video plays on five screens at once. All of it has to happen at the right moment, in the right order, and only when the right conditions are met.

**Maestro** is the engine I built to orchestrate our rooms at Team Break. The name is no accident. In a [previous article](./team-break-architecture.en.md) I described the overall architecture (MQTT, agents, Raspberry Pis, ESP32s, Linux). Here I zoom in on the core of the system: how Maestro turns a stream of heterogeneous events into a coherent experience.

## The conductor plays no instrument

That's the founding principle. The *Room Orchestrator* never drives a relay, a video player or a light directly. It receives **notifications**, decides, then publishes **commands** to agents that know how to talk to the hardware. The video player doesn't know it's showing a dinosaur; the orchestrator doesn't know how to decode H.264.

Four sources of events converge on the same MQTT inbox, `/games/<game>/rooms/<room>/roomorchestrator`:

```
  Players ──► sensors (ESP32, Kincony, I/O panels)
                 │  /c5m1p7/on, {"module":"in2","pin":0,...}
                 ▼
            Electronic Agent ──────────────┐
                                           │
  Agents ──► "video finished", "intro audio ended" ──────┤
                                           │   notifications
  GM ─────► console (milestones, controls, time) ────────┤   (JSON, MQTT)
                                           ▼
  Time ───► delays, scheduled actions ──► Room Orchestrator ──► commands
                                           │                     ├─► audioplayer
                                           │                     ├─► videoplayer
                                           ▼                     ├─► electronics
                                     Agents API (HTTP)           ├─► dmx
                                     PostgreSQL: state,          └─► customagents
                                     milestones, sessions
```

The source of truth for whatever must survive a restart (room state, milestones, session, remaining time) is a PostgreSQL database behind an HTTP API. Everything else (events, commands) travels over MQTT, fire-and-forget.

## The score: the gameflow

What a room does is described in a **gameflow**: declarative JSON, versioned in one Git repository per game and per site. Its vocabulary is small:

- a **notification** is a named event (`digicode_unlocked`, `audio_intro_ended`);
- a **handler** reacts to one or more notifications, optionally filtered by **milestones** and deferred by a **delay**;
- a handler runs **behaviors**, in `position` order;
- a behavior publishes a **command** (or a *bundle* of commands), validates or resets a milestone, ends the room, changes the remaining time, or sends a command to another room.

There are exactly nine behaviors: publish a command, publish a bundle, validate a milestone, reset a milestone, raise a room event (`room_succeed`, `room_failed`), send a command to another room (*dispatch*), set the remaining time, freeze the time display, and release it. That's deliberately short: anything that doesn't fit belongs in a custom agent, not in the engine.

Here's a real excerpt from the Chocolaterie, in Lille. When the audio intro ends (or when the GM forces the matching milestone), three handlers react to the same notification, at t+0, t+1 s and t+6 s:

```json
[
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "position": 1,
    "behaviors": [
      { "behavior": "milestone_validate", "key": "intro_played",         "position": 1 },
      { "behavior": "command_publish",    "key": "audio_background_play", "position": 2 } ] },
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "position": 2, "delay": 1,
    "behaviors": [
      { "behavior": "command_publish",  "key": "r1_pinch_anims_scene1_play",       "position": 1 },
      { "behavior": "command_publish",  "key": "r1_pinch_scene1_audio_play",       "position": 2 },
      { "behavior": "command_publish",  "key": "r1_lights_stainedglass_pinch_on",  "position": 3 },
      { "behavior": "command_dispatch", "key": "chocolate_garden_init_audio_player", "position": 5 } ] },
  { "notifications": ["audio_intro_ended", "milestone_state_change_intro_played_validate"],
    "description": "delai: 5 + 1 (delay between intro and sceno 1 starts)",
    "position": 3, "delay": 6,
    "behaviors": [ { "behavior": "command_publish", "key": "r1_lights_stainedglass_pinch_off" } ] }
]
```

This is how you write a sequence over time: one handler per moment, all on the same notification. The last behavior of the second handler is already getting the next room's audio player ready.

Across the Chocolaterie (five software rooms), that adds up to about 300 command publications, 110 milestone validations, some forty delayed handlers and 136 handlers gated by milestones.

## Inside the room orchestrator

### Start, but only once ready

At startup the orchestrator loads the room settings, its state from the database and the gameflow, retrying every 5 s until everything succeeds. Only then does it subscribe to its topics and report healthy to Docker. The comment in the code explains why:

> Only here does the container turn healthy […]. Signalling before InitializeAsync() would mark the agent healthy while it is subscribed to no topic — the very "Running but inert" false positive.

If the room was mid-game when it restarted (power cut, upgrade), the orchestrator reads the state back from the database and re-arms the countdown where it left off.

A subtle MQTT detail: message handlers are attached *before* `ConnectAsync` is called. MQTTnet starts delivering messages as soon as the SUBACK arrives, and attaching them afterwards lost everything received in between.

### Routing

Every notification carries a `type_id` that decides how it is handled:

```csharp
switch (notification.Type ?? NotificationTypeEnum.RoomEvent)
{
    case NotificationTypeEnum.RoomEvent:
        await HandleRoomEventAsync(notification, scenarioScope); break;
    case NotificationTypeEnum.GameEvent:
        await HandleGameEventAsync(notification, scenarioScope); break;
    case NotificationTypeEnum.GameOrchestratorInstruction:
        await HandleGameOrchestratorInstructionAsync(notification.Source, notification.Key ?? string.Empty, notification.Datas, scenarioScope); break;
    case NotificationTypeEnum.GameMasteringInstruction:
        await HandleGameMasteringInstructionAsync(notification.Source, notification.Key ?? string.Empty, notification.Datas, scenarioScope); break;
    case NotificationTypeEnum.PublishCommandInstruction:
        await PublishCommandAsync(notification.Key ?? string.Empty, scenarioScope); break;
}
```

Game events (the ones that come from players and agents) are only processed while the room is in play (or at the very start of its preparation). A sensor that fires while the GM is tidying the room up triggers nothing. Instructions from the GM and from the game orchestrator always get through.

### Selecting handlers

For a game notification, the pipeline is:

1. **Scenario filter** on the notification itself. A room can run a variant (Halloween, a kids' version), and some elements only exist in one of them.
2. **`only_once`**: some notifications must only count once per session. The list is cleared at every preparation.
3. **Selecting the handlers** that listen to this key, then applying scenario filters and `override`s (a "Halloween" handler masks its standard counterpart).
4. **For each handler**, in order: if it has a delay, it is queued and the loop moves on. Otherwise its **milestone filters** are evaluated (`milestone_validated`, `milestone_not_validated`, combined with `and` or `or`), and then its behaviors run one after another.

```csharp
foreach (var behavior in handler.Behaviors.OrderBy(x => x.Position))
{
    if (!await handlerOptionSelectorScope.ShouldExecuteAsync(notificationKey, behavior.ElementKey,
            behavior.OptionKey, behavior.SelectedItem, CancellationToken.None))
    {
        continue;
    }
    await HandleNotificationHandlerBehaviorAsync(behavior.Behavior, behavior.ElementKey, scenarioScope,
        behavior.Seconds, behavior.NeverAddTime, source);
}
```

The option selector lets a behavior play only for a given value of a session option (typically the language). Command keys can also be built dynamically: `option~lang~fr@_start_video` becomes `fr_start_video` or `en_start_video` depending on the team's language.

One story is a reminder that a declarative format is still code. For a long time, `"filters_behavior": "or"` was evaluated as AND, because a constructor assigned the parameter instead of the property. No gameflow crashed: the affected handlers just only fired when *all* of their conditions were met, which looks a lot like a badly tuned puzzle.

### Concurrency is the real issue

The orchestrator has no inbound queue: each MQTT message opens its own async flow, and several notifications can be processed in parallel. Within a single notification, everything is sequential. This keeps the room responsive (a slow handler doesn't hold up a sensor), but it requires discipline.

First safeguard: state-change instructions (prepare, briefing, start, stop) are serialized by a semaphore. A forced cascade ("prepare the room while it's in play" becomes "stop, then prepare") chains several transitions and must not interleave with another instruction.

Second safeguard, more subtle: **the scenario scope**. Each treatment reads the active scenario once and keeps it to the end. The comment tells the story of the bug behind it (translated from French):

> The risk addressed is not latency but concurrency: […] the notification passed the guard in halloween, the game master switched, and the published command was rejected in standard. The trigger was then only half played.

A half-played effect in a room is a door that opens without the music that announces it, or the other way round. With a single read per treatment, an event is played entirely in one scenario or entirely in the other.

## Time, or rather times

### Delays

Delayed handlers live in memory, in sets sorted by trigger time, polled by a `PeriodicTimer` every 350 ms. Delays are whole seconds: plenty of precision for a stage effect, and a mechanism that stays trivial to reason about.

A handler can also reschedule itself (`"behavior": {"behavior": "while", "delay": 1, "limit": 10}`) to repeat an action, with a bound.

The interesting question is: **when should pending delays be forgotten?**

- **When the room stops**, obviously. But *before* running the `room_stopped` handlers, so that a stop handler can itself schedule a delayed effect.
- **On preparation**, as a precaution.
- **On the emergency signal (fire alarm)**, whatever the room's state. The comment is explicit: "armed before the alarm, they would restart the game when they fall due (sounds, lights, doors) during the evacuation."
- **Not at the end of the game.** That's deliberate: at the Chocolaterie, a win triggers one bundle immediately, then a second one 45 s later, once the team has had time to savour it.

### The countdown, in three clocks

Remaining time lives in three places, and each has its own job:

1. **In the database**, the truth: `start + duration − now + adjustments`. When the gameflow sets the remaining time ("only 5 minutes left!"), what gets stored is a delta. The comment gives the example: *a 60-minute room set to 300 s after 5 minutes of play stores −3000*. GM adjustments are capped at ±320 s per click.
2. **In the orchestrator**, a counter that advances in 5 s steps and fires `room_time_over` at zero. GM clicks **accumulate** until the next step, because several quick clicks land in the same window and a plain assignment would have kept only the last one:

   ```csharp
   public int Tick(int intervalInSeconds, out int appliedGameMasterAdjustmentInSeconds)
   {
       lock (_lock)
       {
           appliedGameMasterAdjustmentInSeconds = _pendingGameMasterAdjustmentInSeconds;
           if (appliedGameMasterAdjustmentInSeconds != 0)
           {
               _remainingTimeInSeconds += appliedGameMasterAdjustmentInSeconds;
               _pendingGameMasterAdjustmentInSeconds = 0;
           }
           _remainingTimeInSeconds -= intervalInSeconds;
           if (_remainingTimeInSeconds < 0) _remainingTimeInSeconds = 0;
           return _remainingTimeInSeconds;
       }
   }
   ```

3. **For the in-room display**, a publisher that computes the seconds left from a UTC **deadline**, re-synced at every step, rather than decrementing a counter. It can be frozen (`remaining_time_display_hold`) for the length of a scene, then released.

   ```csharp
   if (heldSeconds.HasValue) return heldSeconds.Value;
   if (deadlineUtc is null) return null;
   double secondsLeft = (deadlineUtc.Value - nowUtc).TotalSeconds;
   return secondsLeft <= 0 ? 0 : (int)Math.Ceiling(secondsLeft);
   ```

About every 30 s, the orchestrator also publishes a progression update to the outside world (for Archipel's tablets, which interpolate between messages). A coarse cadence is enough: the display does the fine work.

## Milestones, the game's memory

A **milestone** is a named step of progress: "intro played", "keypad found", "chest opened". It is persisted in the database along with its origin (the room, the GM or a scheduled action), and it serves three purposes:

- **gating handlers** ("only play this video if the chest is open");
- **gating hints and scheduled actions**;
- **telling the story of the game** afterwards, in the session timeline.

The GM can force a milestone from the console in two ways. In **quiet** mode only the database changes: useful to fix a state without replaying anything. In **noisy** mode the orchestrator also receives the notification `milestone_state_change_<milestone>_validate` (or `_reset`). Gameflows listen to it next to the "player" notification, as in the excerpt above: the GM can unblock a stubborn puzzle *with* all its effects (light, sound, door), exactly as if the players had solved it.

## Scheduled actions and hints

A dedicated agent, *Scheduled Actions*, listens on the same topic as the orchestrator and keeps its own clock (monotonic, started when the game begins). On each tick (30 s on our sites), it:

1. re-reads the session options ("automatic hints", "automatic wins");
2. takes **a single** snapshot of the room's state (milestones, state);
3. walks the actions in order and runs **at most one action per tick**, never twice in the same session.

An action fires on time elapsed since the start, a delay after a milestone, a delay after a notification, or a remaining time. It can be gated by milestones, and it belongs to a category (`game`, `hint`, `auto_win`) that the GM switches on or off per session. Nothing happens during the first minute of play.

The players' **hint button** follows a different logic. The `hint_system` notification picks the first entry of the hint schedule whose conditions match the progress, then steps through that hint group: first hint, then the next one, then the last one on repeat once the group is exhausted. Every hint is journaled, with this rule written in a comment: "a logging exception must not stop a hint from going out."

## A room's lifecycle

A room goes through eleven states. The basic model is an **asked → confirmed** handshake: the GM console *asks* for a state (in the database), and the orchestrator runs the matching handlers and then *confirms* through the Agents API.

```
 GM UI           GM API / DB            Game Orch.         Room Orch.              Agents
   │ prepare ──►  PrepareAsked
   │              (reset milestones) ──► Prepare ────────► prepare_room
   │                                                       purge delays
   │                                                       room_prepared handlers ──► doors, lights, sound
   │              InPreparation ◄──────────────────────── /prepare/confirm
   │ prepared ──► PreparedAsked ───────────────────────► room_preparation_done handlers
   │ briefing ──► BriefingAsked ───────────────────────► room_briefing_started handlers
   │ start ─────► session created ───► Start + session ─► start_room
   │                                                       room_started handlers ───► intro, countdown
   │                                                       Running, 5 s timer
   │                                                       SessionProgressionInit ──► (Archipel)
   │                  … game: notifications ↔ commands, milestones, hints …
   │                                                       room_succeed | room_failed | room_time_over
   │                                                       then room_ended
   │              Ended ◄──────────────────────────────── /ended/confirm
   │                                ◄── RoomEnded ──────── (chaining, if any)
   │ stop ──────► StopAsked ──────────► Stop ───────────► Stopped first, then
   │              AtRest                                   purge, room_stopped handlers
```

Two nuances, because they're instructive:

- **Pragmatism sometimes beats purity.** For start, stop, briefing and end of preparation triggered from the console, the API confirms the state in the database optimistically ("Resilience fallback: avoid being stuck in StartAsked when orchestrator confirmation is delayed/lost"). A room stuck in "start asked" with six players inside is worse than a momentarily optimistic state. Preparation and the end of the game are still confirmed by the orchestrator.
- **Stopping is unconditional.** The orchestrator switches the room to stopped *before* doing anything else ("Critical safety invariant: room must become Stopped even if a stop handler fails"), then stops the countdown, purges delays, confirms, and finally runs the stop handlers on a best-effort basis. The session ID is only cleared after those handlers, so the end-of-game video montage still knows which session it belongs to.

Some business rules are baked into the guards. For example, a room whose briefing has started can no longer be "prepared": the players are inside, and replaying `room_prepared` would undo the briefing (lights, locks…).

## Orchestrating several rooms: the Game Orchestrator

Some experiences span several rooms. The Chocolaterie chains five: the candy store, the chocolate garden, the elevator, the ice room and the inventing room. Each room has its own orchestrator; above them, the **game orchestrator** acts as section leader.

It is deliberately **stateless**. It knows the order of the rooms, listens on `/games/<game>/gameorchestrator` and relays instructions to the relevant room's orchestrator. When a room ends:

1. the room orchestrator publishes `RoomEnded`;
2. the game orchestrator asks the API to chain. In a single operation, the API stops the ended room's session, resets the next room's milestones and starts it **with the same session ID**: as far as reporting is concerned, the team played *one* game;
3. it publishes the start instruction to the next room.

Rooms also talk to each other during the game through **dispatches**: one room asks another to publish a command, via the game orchestrator.

```json
{ "key": "smoke_on",     "command":  "r2_smokemachine_smoke_on", "recipient_room": "chocolate_garden" }
{ "key": "roller_start", "commands": ["r3_sas_chocolategarden_door_roller_open",
                                      "r3_sas_iceroom_door_roller_close"], "recipient_room": "elevator" }
```

The command is executed by the receiving room, with *its* scenario filters and *its* options.

Finally, the game orchestrator is the entry point for external systems such as Archipel, our site operations platform, which launches rooms from the players' tablets. That's the subject of the [next article](./archipel-orchestration.en.md).

## Electronics in: from `/c5m1p7/on` to `digicode_unlocked`

Sensors don't speak the gameflow's language. An I/O panel publishes `/c5m1p7/on` (controller 5, module 1, port 7); a Kincony board publishes `{"module":"in2","pin":0,"value":"off"}`. The **electronic agent** does the translation:

1. it keeps one MQTT client per electronics broker (one per provider), each with its own reconnection;
2. for each message, it identifies the provider from the topic, ignores heartbeats, builds a resolution key and matches it against the **input proxies** declared in the gameflow;
3. if a proxy matches, it publishes a business notification (`digicode_unlocked`) on the orchestrator's topic, or on several rooms' topics if needed.

In the other direction, a `trappe_unlock` command published by the orchestrator is resolved into **output proxies**, then into a message suited to the board.

An incident showed that translation also has to **filter**. In October, a spurious pulse (`/c5m1p7/on` then `/c5m1p7/off` within the same second) triggered the emergency input. Every input proxy can now carry a `debounce`: a rising edge opens a window (100 ms by default); a release within the window cancels it and is itself swallowed; with no release, the input is confirmed once at the end of the window. The mechanism relies on an injected `TimeProvider`, with no `Task.Delay`, which makes it testable to the millisecond.

## The game master in the loop

The GM console is a web app built with native Web Components, no framework: room control, countdown, milestones, controls, cameras, push-to-talk to the room's speakers, a technical page to test every I/O point, a tests page, and session timelines.

For live updates I went for simplicity. A single server-side poll per room (every 1.5 s, started by the first subscriber and stopped by the last) reads the database and pushes deltas over **SSE** (`milestones`, `state`, `timer`, plus a `heartbeat`). The comment sums up the argument: this mechanism "captures every source of change (GM, players, scheduled actions, Room Orchestrator confirmations) since everything goes through the database." No need to subscribe the browser to MQTT or to rebuild state from events.

GM controls ("open door 2", "restart the video") are lists of actions run in order. All of them are attempted even if one fails, and the result has three states: success, partial action, impossible action. During a game, the GM needs to know *what* worked, not just *whether* it worked.

## Safety, outside the score

Emergencies don't go through the gameflow. A dedicated agent with a minimal critical path (no HTTP, no database, infinite MQTT reconnection, `restart: always`) relays the fire-alarm system: doors open, lights on, sound cut, and a latch published as *retained* that every restarting agent reads back immediately. On the orchestrator side, the emergency signal purges pending delays whatever the room's state. Above all, life safety (the door maglocks) stays hard-wired on the fire-alarm side, outside software. Maestro follows the alarm; it doesn't replace it.

## Tracing and testing

Every game leaves a timeline: state changes, milestones validated or reset (with their origin), published commands, GM controls used, clicks on the technical page. The GM action log (mic, volumes, cameras) deliberately has no foreign key, "so that neither a TRUNCATE … CASCADE… nor the purge… erase the log." *Reporters* push finished sessions to our head office (outside opening hours) and to the booking tool.

On the testing side, the room orchestrator has its own suite (184 tests), among the roughly 1,800 in the repository. Every gameflow also ships a **test protocol** (`_tests/*.json`), never read by the game, validated in CI and replayed from the console: that's where the Chocolaterie's 312 I/O points get checked one by one.

## What I take away from it

1. **The conductor plays no instrument.** The orchestrator decides and publishes; agents execute. That's what lets us replace a video player, an electronics board or a light without touching the engine.
2. **Declarative doesn't mean bug-free.** A gameflow is code: it deserves a documented format, tests and CI.
3. **Concurrency is the real issue.** Processing events in parallel keeps the room responsive, as long as the context (scenario, options) is frozen for the duration of a treatment.
4. **Time is plural.** Truth in the database, a counter in memory, a deadline for the display, interpolation on the client: each clock has its job, and none should pretend to do the others'.
5. **Know when to forget.** Purging delays (on stop, on preparation, on alarm, but not on a win) is a staging decision as much as a technical one.
6. **Confirm, but never get stuck.** The asked → confirmed handshake is the rule, and optimism a deliberate exception when players are waiting.
7. **Safety short-circuits everything.** It has its own agent, its own path, and the last word.

A room is a few hundred notifications per game, a few hundred commands, a handful of milestones and a countdown. When they're well orchestrated, players see none of it. They see a door opening at the exact moment the music swells.
