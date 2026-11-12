> **Alternative titles (pick one, then delete this block before publishing)**
>
> 1. Five rooms, three teams, one Pinch: the engineering behind La Chocolaterie
> 2. Invisible walls and null bytes: what La Chocolaterie taught me
> 3. Godot, MQTT and eight buttons in an elevator: a La Chocolaterie post-mortem

# La Chocolaterie: behind the scenes of a five-act escape game

*A platformer played on a real set, an eight-player Guitar Hero inside an elevator, and teams that have to make it through five rooms without ever getting stuck. A post-mortem.*

---

Of all the games I have worked on, **La Chocolaterie** is the one that gave me the hardest time. It is not one room but **five chained rooms**: the *Candy Store*, the *Chocolate Garden*, the *Elevator*, the *Ice Room* and the *Inventing Room*. Players are guided by Pinch, a small character who shows up in the videos, comments on what they do, and eventually becomes… the hero of a platform game. And to make things harder, up to three teams can play it at the same time.

The first lines of code date back to summer 2024. It took a lot of iterations, many evenings on site and plenty of fixes to get to a game that runs every day without surprises. Today the room is a success, and the players say it better than I can:

<!-- Google review screenshots -->
![Google review](./images/google-review-1.png)
![Google review](./images/google-review-2.png)

In this post I go back over the three workstreams that kept me busiest: **the platformer**, **the Guitar Hero**, and **managing the flow of players** through the five rooms. And what I took away from them.

## La Chocolaterie by the numbers

| | |
|---|---|
| **5** | chained rooms, each with its own orchestrator |
| **3** | teams playing at the same time, at most |
| **312** | declared electronic inputs/outputs (sensors, relays, doors, fans…) |
| **78** | gameflow JSON files |
| **275** | notification handlers |
| **615** | commands (audio, video, electronics, custom agents) |
| **341** | gameflow versions released so far |
| **13** | game-specific .NET agents, plus a Godot game and an ESP32 firmware |

## The architecture in a nutshell

All our games run on **Maestro**, the platform I built to drive our rooms. The idea: specialised agents talking over **MQTT**, and a **gameflow** described in JSON rather than in code.

```
                    ┌──────────────────────┐
                    │   GM UI  /  GM API   │   game masters
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Game Orchestrator   │   chains the rooms
                    └──────────┬───────────┘
                               │ MQTT
     ┌───────────┬─────────────┼─────────────┬──────────────┐
     ▼           ▼             ▼             ▼              ▼
  Room orch.  Room orch.   Room orch.    Room orch.     Room orch.
  R1 Candy    R2 Garden    R3 Elevator   R4 Ice room    R5 Inventing
     │
     ├── Electronic agent ── I/O boards (sensors, relays, doors)
     ├── Audio player       (Python / pygame)
     ├── Video player       (mpv, on Raspberry Pi)
     └── Custom agents      (Guitar Hero, platformer, pump machine…)
```

- Each **room orchestrator** runs its room's gameflow: it receives notifications (a sensor, the end of a video, an agent reporting success), applies filters (is this milestone validated?) and publishes commands.
- The **electronic agent** translates the raw format of the I/O boards (something like `/c3m4p4/on`: controller 3, module 4, pin 4) into business keys such as `fanbtns_1_pressed`. That mapping, the *electronic proxies*, also lives in the gameflow.
- The generic agents (audio, video, electronics, orchestrators, GM interface) contain **no game-specific code at all**. Everything specific to La Chocolaterie lives either in the gameflow or in a dedicated agent.

That data-driven choice turned out to be decisive. The 341 gameflow versions are 341 adjustments (a hint coming too early, a video running too long, a missing filter) that I shipped **without recompiling anything**.

## Workstream #1: a platformer played on a real set

### The concept

In the Ice Room, Pinch becomes the hero of a platform game. But the goal is not to display a game: it is to create **the illusion that Pinch is really moving around the room**. In production, the background is black and **only Pinch and the bubbles that simulate the wind are displayed**. Everything else is physical: Pinch moves along the room's **pipes**, and the fans that lift him are **real fans**, installed in the room.

Players control Pinch with physical controls (left, right, jump) wired to the room's electronics. To get through some sections, they have to switch fans on or off: the real airflow and the in-game wind zone start together. In level 2, they have to **move pipes** in the room: their position, read by sensors, opens or closes the matching walls in the game.

### The stack

- **Godot 4.4 .NET edition**, scripts in **C#**, *GL Compatibility* renderer;
- **two Raspberry Pi 4** boards (Linux arm64), one per level, started with `-level 1` or `-level 2`;
- every input comes in over **MQTT**: in a release build, the keyboard does not exist.

### Invisible level design

The first real problem is geometric. The level does exist in the game, with its walls and platforms, but it is never displayed in production: **the colliders have to match, as closely as possible, pipes that the players can see but the game cannot**. Shift a virtual wall by a few centimetres and Pinch floats in mid-air or goes straight through a pipe.

So calibration happens on site. A debug build displays the whole level, walls included: that is what lets me check, in the room, that the game's walls line up with the physical pipes. The fan zones, on the other hand, were only visible in debug, because their display was toggled at compile time with `#if DEBUG`. I turned that into a launch option that works directly on the Pi's release build:

```bash
./plateformer.sh -level 2 -debug-zones
```

Wind zones show up in red, limits in green, emission areas in blue, and you adjust against the real fans. That is also how a "fridge wall" appeared in level 2, a year after the first version: the set had its own opinion on the matter.

### Fan physics

In the game, a fan is a particle emitter (the bubbles) and two `Area2D` nodes: a **wind zone** that pushes Pinch, and a **limit zone** at the top where he hovers. To avoid a character frozen in mid-air, he gently oscillates around that point:

```csharp
if (_isInVerticalFanLimit)
{
    // Top of the airflow: subtle oscillation
    float oscillation = Mathf.Sin(_oscillationTime * OscillationSpeed) * OscillationStrength;
    velocity.Y = oscillation;
}
else if (_isInVerticalFan)
{
    // Inside the airflow: upward push
    velocity.Y += -350 * (float)delta;
}
else if (!IsOnFloor())
{
    velocity.Y += 1000.0f * (float)delta; // gravity
}
```

On the electronics side, I wired the real fan **as a reflex**: the button's proxy drives the fan's output on the board directly, with no round-trip through the orchestrator or the game. The fan starts blowing the instant the button is pressed, and the game receives the same information in parallel.

### Hints that come from the game itself

When Pinch gets stuck at the foot of a fan, he enters a `DetectionZone`. The first time, the game publishes a notification (`plateformer_ventilation_pinch_stuck`) and the gameflow plays an audio hint. It is a contextual hint, driven by **the character's position** rather than a simple timer. Game masters also have unblocking commands (`solve_fans`, validating a level…).

### The intro video: frames instead of a video

Godot only plays Ogg Theora natively, and on a Pi 4 the result did not convince me. So the intro was split into frames:

```bash
ffmpeg -i intro.mp4 -vf scale=960:540,fps=15 frames2/frame_%04d.jpg
```

Later I noticed that out of 369 frames, **80 were exact copies** of the previous one. By merging each run into a single `SpriteFrames` entry, with a duration equal to its length, I cut **22% of memory and loading time**, with identical playback.

### The real final boss: MQTT

Godot has no native MQTT client. So the game ships a **minimal client over WebSocket**, ported into the project. It worked for months… until I went looking for why the room occasionally got stuck. Auditing it, I found quite a collection:

- **No reconnection, ever.** The state machine only retried when `_reconnectTimer > 0`, but after a failure that timer was `0`. If the Pi booted before the broker, the game stayed deaf until restarted.
- **No connection at all in quiet mode**: the `ConnectToUrl(...)` call was nested inside an `if (_verboseLevel != 0)`.
- **Packet encoding**: the `PUBLISH` buffer size was computed as `sz/127 + 1` instead of the real variable-length size. From 255 bytes on, an extra null byte went out after the packet and the broker dropped the connection. The monitoring ping was 147 bytes, so not that far off. The remaining length of `CONNECT`/`SUBSCRIBE` was written on a single byte, and the 120 s keep-alive was announced as 124 s because of an `OR 0x3C`.
- **One packet processed per frame**: a burst of N messages was applied N-1 frames late.
- **A poisoned packet**: the receive buffer was not cleared on disconnect. A truncated packet stayed at the head of the buffer and made every new connection fail before the `CONNACK`, over and over.
- **The level-complete message got lost**: `levelN_succeed` was sent with QoS 0. On a dead link not yet detected (up to ~30 s), the message went nowhere, the level was already hidden, and the orchestrator never found out. **The room was stuck.**
- And to top it off, a `"BB"` *last will* published on a `last` topic, left over from a test, plus **three random client ID generators** stepping on each other.

The fix started as a specification with measurable acceptance criteria, followed by a series of small PRs:

- **exponential backoff** (1, 2, 4, 8, then capped at 10 s), a 5 s timeout on every transient state, and dead-link detection after two missed `PINGRESP`s;
- the level-complete message published with **QoS 1**, kept in an *outbox* until its `PUBACK` and re-sent on every reconnection. A duplicate is possible, a loss no longer is;
- a real "offline" *last will* on the monitoring topic;
- a **stable client ID**, `choco-r4-plateformer-<host>-level<N>`. The level is part of it, so that two Pis left with the default hostname do not keep kicking each other off the broker.

Above all, all that logic was extracted into **pure classes** (`MqttReconnectionPolicy`, `MqttPacketCodec`, `MqttCriticalNotificationOutbox`…), tested with xUnit **without instantiating the Godot engine**, using injected time. A thousand consecutive connection failures run in a few milliseconds.

Along the way, another sneaky bug: the player was instantiated as soon as the level loaded, invisible but **physically active**. A ghost Pinch could reach the exit before the game had even started. It is now disabled (`ProcessMode.Disabled`) outside a running level.

## Workstream #2: an eight-player Guitar Hero in an elevator

### The brief

In the elevator, **eight buttons** are arranged in pairs next to the screens. A video with a soundtrack scrolls notes, and players have to press at the right moment, sometimes holding for long notes. The spec fitted in a few lines, some of them heavy with consequences:

- difficulty adapted to **the number of players**;
- a game that becomes **more forgiving** after each failure, with a limited number of restarts;
- an on-screen dial showing how many errors are still allowed;
- a missed note counts as an error;
- video and audio are handled by other agents.

And one question was left open in the original README: *"How do we test the game? Do we have to be in the room, with several people?"*

### Time, enemy number one

The notes are described in a `Scenarios.json` file (start and end in milliseconds on the video, `solo` for the red solo notes), validated at startup: a button that does not exist or a note ending before it starts makes the load fail. That is not the hard part. The hard part is that **the game and the video do not share a clock**. The game starts its stopwatch when it sends the play command, and the video starts… a bit later.

One constant in the code shows how I first handled it:

```csharp
// In reality all notes lag behind their schedule, cause to be identified.
internal const int REALITY_LAG_SO_TIMELAPSE_TO_ADD = 1000;
```

It is an empirical compensation, measured on site. It works, but open-loop. Since then, the video player can report **the actual start of playback**, and the game can **re-anchor its clock** on it. That closed loop is ready but still dormant: switching it on means recalibrating the constant, and on site you do not touch something that works without a test session.

### A single game loop, no locks

Button presses arrive over MQTT, the missed-note timer runs on its own thread, and start/stop commands come from the gameflow. The first versions had **races**, of course. The current one rests on a simple model: every event goes into a `Channel<T>` and **a single consumer** mutates the game state. Feedback (sounds, dial, Pinch's voice) goes into a second channel drained by a dedicated pump, so the loop never blocks on the network.

```csharp
private void Dispatch(GameEvent gameEvent)
{
    switch (gameEvent)
    {
        case StartEvent: HandleStart(); break;
        case StopEvent:  HandleStop();  break;
        // An event stamped in a previous round carries a time from the old clock:
        // applying it to the fresh notes would mark notes missed or score errors.
        case ButtonPressedEvent e when e.Round == _round:
            HandleButtonPressed(e.ButtonId, e.TimestampMs);
            break;
        case TickEvent e when e.Round == _round:
            HandleTick(e.TimestampMs);
            break;
        // ...
    }
}
```

That **round** number (`_round`) fixed a nasty bug. A press queued just before a restart was processed just after it, with a timestamp from the old clock (say 60,000 ms), and applied to brand-new notes. A single tick could then mark seven notes as missed at once and trigger a bogus restart.

### Game rules, escape-room edition

A few choices that look minor but made all the difference:

- **One press resolves at most one note**: the earliest pending note on that button. An older version could resolve up to seven, which skewed the score and caused unfair restarts.
- **A wide window**: ±600 ms, plus a 250 ms tolerance. This is not Expert mode. The players are not gamers, there are eight of them, and they are discovering the buttons.
- **A note is only flagged as missed after a 350 ms grace period** beyond its window, so a late press does not get overtaken by the check. The check runs every second and flags at most seven notes per pass, so the dial goes down gradually.
- **Difficulty depends on the number of players** (provided by the session): *easy* up to 2, *medium* for 3 or 4, *high* beyond.
- **The game cannot be lost forever**: 15 errors allowed on the first try, 30 on the second, the tolerance widens by 200 ms on each restart, and on the last allowed try the tolerance jumps to 1,500 ms with an unreachable error cap. **Victory is guaranteed.**

That last point sums up a rule I consider fundamental: **in an escape game, a challenge must never be a dead end.** The team has to get out of the elevator, feeling that they earned it.

### Testing without eight people in an elevator

The answer to the README's question is called `GuitarHero.AutomaticTest`: a tool that **replays timestamped press/release scenarios** over MQTT, exactly like the real buttons. Some scenarios are "perfect", others deliberately sloppy, others are supposed to trigger a restart. Start the game, start the scenario, watch the dial. It is not a unit test in the strict sense, but it is infinitely more reproducible than "let's go back in there with four people tonight".

## Workstream #3: moving players through five rooms

### Three teams at once

La Chocolaterie does not host one team at a time. **Up to three teams can play simultaneously**: one in room 1, one in room 3, one in room 5. That is how the room normally runs, and it happens very often at weekends.

That changes the nature of the problem entirely. A room is no longer an isolated game, it is a stage in a pipeline: each team moves one room behind the previous one, and that gap has to hold from start to finish. When a team falls behind, the next team catches up with it.

The architecture lends itself to this: each room has its own room orchestrator, with its own gameflow and its own state. Three games run in parallel without stepping on each other, and game masters have a multi-room view to follow them.

### Chaining the rooms

Each room has its own **room orchestrator**, but a team plays a single game. The **game orchestrator** ties it together: when a room publishes `room_succeed`, it records the end, finds the next room and starts it **with the same session**. Everything starts in the first room (preparation, briefing, launch), and the following rooms are started by the chaining. The session follows the team, not the room: its name, its language and its number of players (the one that sets the Guitar Hero difficulty) travel from room to room, even with three teams in the building.

Physical transitions are scripted too. In the elevator's airlock, for instance, the welcome video **loops until the players have closed the door**:

```json
{
  "notification": "r3_sas_play_ended",
  "description": "End of the scene, replay it if the players haven't closed the door",
  "filters": [{ "behavior": "milestone_not_validated", "key": "r3_door_closed" }],
  "behaviors": [
    { "behavior": "command_publish", "key": "r3_sas_entrance_video_play" },
    { "behavior": "command_publish", "key": "r3_sas_entrance_audio_play" }
  ]
}
```

### The time budget: hints first, then safety nets

This is the real design challenge. In a single room, a team running late simply ends without finishing. In a five-act journey, **a team stuck in room 2 will never see rooms 3, 4 and 5**, which are the heart of the experience. And with three teams in play, it also holds up **the team coming in behind it**. So each room's time budget is not a nice-to-have, it is an operational constraint.

So each room has a time budget, handled in two layers in the gameflow:

1. **Progressive automatic hints**, triggered by *scheduled actions* relative to a milestone. For example, 4 minutes after the badge is placed in the cash register, a hint about the jars; 8 minutes after, a hint about the taps; and so on. Each hint is conditional: if it is no longer useful, it does not fire.
2. **Forced validations**, as a last resort, that move the room forward on the players' behalf, in a scripted way.

| Room | Safety nets (time since trigger) |
|---|---|
| R1 – Candy Store | 2, 8, 23 and 25 min |
| R2 – Chocolate Garden | 12, 22 and 30 min |
| R3 – Elevator | 2 and 18 min (plus the unlosable Guitar Hero) |
| R4 – Ice Room | hatch opened at 10 min, platformer validated at 40 min |
| R5 – Inventing Room | 9, 12 and 22 min |

In the Ice Room, if the keypad code has not been found after 10 minutes, Pinch opens it himself. The players do not experience a failure, they experience a plot twist:

```json
{
  "description": "After 10 minutes, open the duct",
  "trigger_type": "room_event",
  "trigger_key": "room_started",
  "delay": 600,
  "conditions": [{ "behavior": "milestone_not_validated", "key": "r4_digicode_unlocked" }],
  "behaviors": [
    { "behavior": "command_publish",    "key": "trappe_unlock" },
    { "behavior": "milestone_validate", "key": "r4_digicode_unlocked" },
    { "behavior": "command_publish",    "key": "digicode_unlocked_bypinch_audio_play" }
  ]
}
```

On top of that, game masters see milestones in real time and can send hints, validate a challenge or unblock a fan from their interface. Automation handles the general case, and a human stays in control.

### Stopping cleanly is gameplay too

Something I underestimated at first: **stopping a room**. A stop from the game master has to open every door and hatch, fade Pinch's videos to black, hide the platformer, cut the effects. Several recent gameflow fixes are only about that, for example "the GM stop hides Pinch and the platformer", or "GM validation of level 2 hides the level and Pinch". And since teams follow each other closely, any state left over at stop time quickly turns into a bug for the next team.

## Operations: the room has to run without me

An escape game means several sessions a day, every day. Here, up to three at once. A few building blocks you do not see but that matter:

- **Safety.** The emergency agent listens to the fire alarm input and puts the room in a safe state on the software side (sound off, lights on, unlocking). Life safety itself **stays hard-wired, outside the software**. The audio player keeps its own watch on the alarm: there are two independent paths to silence, and a single, explicit path back. A reset is refused while the alarm is still active.
- **Power cuts.** Autonomous recovery of a room after a power cut has its own test bench, on a VM that gets switched off brutally.
- **Deployments.** The platformer deploys to the Pi with one command, with a backup of the previous version and a documented rollback procedure.
- **SD cards.** During a broker outage, MQTT warnings added thousands of log lines a day to the SD card of a Pi that runs around the clock. I set up `logrotate` in `copytruncate` mode, so the game does not need restarting.

The latest step so far: a **systematic hardening** phase. I set up a multi-agent "software factory": a specification with testable acceptance criteria, a human approval gate, the implementation, then an adversarial verification. It let me handle dozens of small latent defects as short, reviewable PRs. The platformer's MQTT client and the Guitar Hero were the first to benefit.

## What I take away from it

1. **The physical world lies.** Media latency, bouncing sensors, a set that does not match the plan. Every timing constant has to be measured on site, and documented as such.
2. **A challenge must never be a dead end.** Widening tolerance, progressive hints, scripted forced validations: players must always move forward, and feel that they made it happen.
3. **Think pipeline.** With three teams in parallel, a room is no longer a game but a stage: how long it takes to get through matters as much as what is in it.
4. **Data-driven or nothing.** 341 gameflow versions means 341 times I did not have to recompile an agent to shift a hint by two minutes.
5. **The "simple" building blocks are the ones that break.** The platformer's gameplay was stable early on. A few hundred lines of MQTT client were what kept getting the room stuck.
6. **Make testable what isn't.** A game engine, an elevator, eight buttons: none of that runs in CI. A reconnection policy, a packet codec or an event loop, on the other hand, can. Extract those parts, and you can test them.
7. **A client that disappears should say so.** Last will, stable IDs, explicit logs: in production, knowing *who* went down is worth as much as bringing it back up.

La Chocolaterie was a real challenge, as much in game design as in engineering. Seeing teams walk out of the Inventing Room with a smile today, and then write about it in their reviews, is the best reward for all those hours spent aligning invisible walls and hunting down null bytes.

<!-- Google review screenshots -->
![Google review](./images/google-review-3.png)
