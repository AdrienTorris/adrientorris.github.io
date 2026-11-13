# Our lobby is a Maestro room (with no game in it)

*How we moved the lobby of our Lille venue onto our escape game orchestration platform, writing almost no code along the way.*

At Team Break, the **lobby** is the shared space of each venue. It's where players wait before their mission, and where game masters (GMs) gather them for the **brief**, the story set-up before they enter the room. The lobby isn't a game room, but it's the first thing customers see and hear. It has its own music, lights in front of each themed room, and even a small hidden game.

Until this autumn, all of that ran on the supplier's legacy controller: a Raspberry Pi running Node-RED, wired to two I/O panels. It did the job, but it lived apart from our tooling: no GM interface, no per-room brief button, nothing under version control.

Our rooms, meanwhile, are moving one by one onto **Maestro**, our in-house platform. Hence a simple idea on paper: **treat the lobby as a full-fledged Maestro room**. The Lille lobby led the way and is now in service. The Paris 9, La Défense and Aéroville lobbies are being installed. This post covers how we did it in Lille, what the platform gave us for free, and the pitfalls we hit along the way.

---

## Maestro in two minutes

Some context for the rest of the post:

- **One stack per room** (the `maestro-room` Debian package, Docker Compose), written in .NET. It includes an API and a web interface for GMs (the **GM UI**), a **room orchestrator** that runs the room's scenario, an **electronic agent** that translates messages from the I/O panels, and a few other agents.
- Everything talks over **MQTT**.
- Around the stack run a few **host agents**: our MQTT-driven audio player (MQTTaudioplayer), `blinker-agent` for light sequences, and `io-status-agent` to handshake with the I/O panels.
- A room's behaviour is **not code**. It's a **gameflow**: a set of JSON files in one GitHub repository per room, loaded by the orchestrator at startup (and cached locally so it keeps working offline).
- A gameflow is mostly a vocabulary:
  - **commands** (play a sound, switch an output on);
  - **notifications** (a GM button, an electronic input, the end of a sound);
  - **handlers** that chain *behaviors* when a notification arrives, subject to **filters**;
  - **milestones**, i.e. steps that are validated or not.
- GM UI data (rooms, buttons, categories) is imported through **versioned `_datas` files**: one initialisation, then patches.

The golden rule of the `maestro-agents` repository: **no game-specific code, ever**. Anything specific to a room lives in its gameflow.

---

## The requirements

The scoping document fit in a few lines:

1. A GM UI with **one button per room**: "Launch the Magic School brief", "Launch the Haunted Room brief"…
2. Each button **plays the right brief audio**. When it ends, we go back to the **default mode**: background music that runs all day.
3. **Later on**, light effects for each brief.
4. The **hotel bell** in front of the Haunted Room. A customer rings it: a bell sound, then a special ambience and the façade lights going wild.

The Lille lobby serves 11 rooms and 13 briefs (two rooms have a variant: Halloween for Magic School, Asylum for Prison Break). It has two I/O panels: one input (the bell) and 17 outputs (logos, windows, corridors, spotlights).

The first conclusion of the scoping work was the pleasant surprise of the project: **phase 1 required no development at all**. Everything already existed in Maestro and in the audio player. We needed a gameflow and a server. We touched platform code only once, for the Start / Stop states (more on that below).

---

## Architecture

```
 Front-desk GM station (GM UI)
        │  "Prison Break / Launch" button
        ▼
 GM API ──MQTT──► room orchestrator (lobby gameflow)
                     │
                     ├──► MQTTaudioplayer ──► lobby amplifier
                     │         ▲
                     │         └── "lobby_brief_ended" when the file ends
                     │
                     ├──► blinker-agent ─────┐  light sequences
                     │                       ▼
                     └──► electronic agent ──► IO broker ◄──► panels c0 / c1
                                 ▲                                │
                                 └──────── hotel bell ◄───────────┘
```

The lobby has **its own server**: the Raspberry Pi 4 that used to run the legacy controller, reflashed with Raspberry Pi OS Lite 64-bit (Debian 13 "Trixie"). The `maestro-room` stack ships an `arm64` build and needs 4 GB of RAM. We ruled out hosting the lobby on one of the rooms' servers (a second audio output would have been enough). With a dedicated server, updating a room never silences the lobby, and the lobby never puts a running game at risk.

---

## A room… without a game

A Maestro room is a state machine: preparation, briefing, Start, end of game, countdown. The lobby has none of that. First version, purely declarative: switch it all off.

```json
{
  "key": "lobby",
  "name": "Lobby",
  "automatic_end_enabled": false,
  "is_prepare_enabled": false,
  "is_start_enabled": false,
  "is_state_machine_active": false,
  "is_session_evaluation_popup_enabled": false,
  "is_add_time_proposal_enabled": false
}
```

This works because two Maestro behaviours, checked in the code before we committed to the approach, played in our favour:

- the GM API runs a button **without checking that a session is in progress**;
- the orchestrator routes a button's notification straight to the handlers. With the state machine disabled, there's no state check left at all.

So the GM buttons work at any time.

### Then came Start and Stop

A week after scoping, the teams asked for one button that starts the default mode and one that shuts everything down. Those are standard Maestro states. But without a preparation step, the Start button stayed **permanently greyed out**: no path through the state machine led to it. The irony: the orchestrator already accepted that Start, since it only checks state when the state machine is active. The block was purely in the domain model.

Hence our single code change (maestro-room 1.17.3):

```csharp
private bool IsStartWithoutPreparationAllowed => !IsPrepareStateEnabled && !IsStateMachineActive;

public bool IsStartAvailable()
{
    if (!IsStartStateEnabled)
    {
        return false;
    }

    // Room without preparation (lobby, sessionless room): only Start and Stop exist, and Start begins
    // from rest. The previous session's Started survives the Stop: it doesn't block the next cycle;
    // only a session in progress does.
    if (IsStartWithoutPreparationAllowed)
    {
        return StateId == (int)RoomStateEnum.StartAsked
            || ((StateId == (int)RoomStateEnum.Undefined || StateId == (int)RoomStateEnum.AtRest)
                && !GameSessionId.HasValue);
    }

    // … historical path for rooms with preparation, unchanged
}
```

The change is deliberately narrow. Only rooms with preparation **and** the state machine disabled are affected; every other room behaves exactly as before. On the data side, a patch enables Start and Stop, disables briefing (on by default) and sets the duration to **720 minutes**, so the GM UI countdown never reaches zero during the day. Automatic end stays disabled: nothing happens at 0.

Start and Stop then become two plain `game_event_handlers`:

| Event | What the gameflow does, in order |
|---|---|
| `room_started` | validates the `lobby_running` milestone, releases the locks, restarts the ambience from the top, stops light sequences, puts the Haunted Room façade at rest, starts the Prison Break corridor alternation |
| `room_stopped` | resets the milestones **then** stops all sounds, stops every sequence, switches off the lights of the 5 rooms that have some |

The order of the Stop matters. If the player publishes an end-of-track message during the shutdown, no lock is valid any more and nothing gets restarted. The orchestrator also cancels pending delayed handlers before running the Stop handlers.

When the server boots, the orchestrator does exactly one thing: send the track catalogue to the audio player, 10 seconds after connecting to the broker. **The lobby stays silent until Start.** That was an explicit choice from the teams, and it's the most predictable behaviour after a power cut.

---

## Playing the right music for each brief

This is the heart of it. The GM presses "Launch": the ambience fades out, the brief plays, the ambience comes back.

```
GM button "Koh Lanta / Launch"
  → notification lobby_brief_koh_lanta_asked
  → room orchestrator: lock lobby_brief_running
                       lobby_ambiance_fade_out    (ambience → 0 in 3 s, without stopping)
                       lobby_brief_koh_lanta_play (solo, at_end_message = lobby_brief_ended)
  → end of file: the player publishes "lobby_brief_ended" to the orchestrator
  → room orchestrator: lobby_ambiance_fade_in     (ambience → 0.6 in 3 s, from where it was)
                       lock released
```

The brief command:

```json
{
  "key": "lobby_brief_prison_break_play",
  "recipients": [ { "type": "audio_player" } ],
  "format": "audio_player_play",
  "datas": [
    { "key": "track",          "value": "brief_prison_break" },
    { "key": "volume",         "value": "1.0" },
    { "key": "solo",           "value": "true" },
    { "key": "at_end_message", "value": "lobby_brief_ended" }
  ]
}
```

And the return to default mode, shared by all 13 briefs:

```json
{
  "key": "lobby_brief_ended",
  "notification": "lobby_brief_ended",
  "behaviors": [
    { "behavior": "command_publish", "key": "lobby_ambiance_fade_in", "position": 1 },
    { "behavior": "milestone_reset", "key": "lobby_brief_running",    "position": 2 }
  ]
}
```

A few properties of the audio player do all the work:

- **A fade doesn't stop the track.** The ambience keeps running at zero volume during the brief and resumes exactly where it was, instead of starting over.
- **`solo` within the `lobby_brief` group.** A brief launched during another one cuts the first. The cut brief doesn't publish its end; the second one will bring the ambience back.
- **A deliberate stop never publishes `at_end`.** So the "Stop the brief" button brings the ambience back itself.
- **The end message is plain text**, which the orchestrator treats as a notification key, **even outside a session**.

### The obvious solution, and why we didn't use it

The audio player has a command built for exactly this: `notify`. It lowers the other tracks (*ducking*), plays the sound, brings them back at the end, and can even queue sounds. It was the simplest option, with no round trip through the orchestrator.

Replaying it in the player's integration harness, we found a flaw. **A `notify` stopped by a command (`stop`, `stopall`) is never cleaned up**: the lowered volumes aren't restored, and the track stays flagged as an active notification. After a "Stop the brief", the ambience stayed at 0. Worse: at the end of **every subsequent brief**, it was never brought back, because it was "still lowered by another notification".

So we went with an explicit fade plus `play` with `at_end`. The cost is known and documented: bringing the ambience back goes through the orchestrator. If it restarts right as a brief ends, the ambience stays at 0 until someone hits the "Stop everything and restart the ambience" safety button. The player fix has been proposed separately; the rooms will benefit from it too.

### Small format traps

| Trap | Effect | Workaround |
|---|---|---|
| `audio_player_play` **doesn't forward** the `replay` option to the player | No way to say "start the ambience unless it's already playing" | A `raw` command, with the player's JSON written by hand |
| A `raw` command without `mqtt_topic` goes to `/<game>/raw` | Message lost, nobody listens on that topic | Explicit topic: `/games/lobby/rooms/lobby/audioplayer` |
| `_electronic_agent_settings.json` is mandatory | The gameflow won't load ("No providers"), even without electronics | A provider declared on the panels' broker |
| GM button descriptions are capped at 320 characters in the database | Button missing if it's too long | Short descriptions |

The `raw` command in question:

```json
{
  "key": "lobby_ambiance_ensure",
  "format": "raw",
  "datas": [
    { "key": "mqtt_topic",   "value": "/games/lobby/rooms/lobby/audioplayer" },
    { "key": "mqtt_message", "value": "{\"command\":\"play\",\"data\":{\"track_id\":\"lobby_ambiance\",\"loop\":-1,\"fade_in\":3000,\"volume\":0.6,\"replay\":false}}" }
  ]
}
```

---

## Background audio: one file, on purpose

The default mode is about twenty tracks played back to back. We could have chained them one by one with `at_end`. But then **every transition would have depended on the orchestrator**: one restart, and the lobby goes quiet in the middle of the day.

So we did the opposite. The tracks are converted, loudness-normalised to −16 LUFS and **concatenated into a single WAV**, which the player loops forever (`loop: -1`). As 16-bit / 44.1 kHz WAV, it **streams from disk** block by block, using about 540 KB of RAM regardless of length. On a Raspberry Pi, that matters. The ambience now depends on the audio player alone.

The player's catalogue fits in one initialisation command (18 tracks, two groups):

```json
{ "key": "track_1", "value": "key~lobby_ambiance|path~/medias/audios/lobby_mix.wav|group~lobby_ambiance" },
{ "key": "track_2", "value": "key~brief_koh_lanta|path~/medias/audios/kl.wav|group~lobby_brief" }
```

The downside is accepted: **changing the playlist means re-rendering the mix**, copying it to the same place, then clicking "Stop everything and restart the ambience". That limit is what pushed us towards proper playlist management, now being tested in Lille (see "What's next").

---

## Light ambiences

The lobby's 17 outputs come from the legacy controller's I/O map and are declared as **proxies** of the electronic agent: 36 of them, one `on` and one `off` per I/O. Each one can be toggled from the technical page as soon as the server is installed. A handler switches an output on with a plain `command_publish` of its key (`lobby_lost_wall_light_on`…): the loader builds the command and routes it to the electronic agent.

For **sequences** (blinking, alternation, timers), we use `blinker-agent`, which plays timed series of messages to the panels. The Prison Break corridor's normal mode, a blue / red alternation every 2 seconds, looks like this:

```json
{
  "command": "sequence",
  "datas": {
    "ID": "lobby_prison_break_rest",
    "loop": -1,
    "sequence": [
      { "messages": ["/c0m1p2/off", "/c0m1p1/on"],  "time": 2000 },
      { "messages": ["/c0m1p2/on",  "/c0m1p1/off"], "time": 2000 }
    ]
  }
}
```

What we learned about blinker-agent, the hard way or by reading its code:

- `time` is the delay **before** the step, and defaults to 1,000 ms: the first step needs `time: 0`.
- In a loop, the first step follows the last one with no delay, so each step carries its own delay. For the alternation, blue is switched on just before the sequence starts.
- Sending a sequence with the same `ID` **restarts it from the top**. We rely on that: the Haunted Room brief and the bell share the same `ID`.
- `stop` **freezes** outputs in their last state. Every stop is therefore followed by an explicit state ("façade at rest": windows off, logo on).
- `stop_all` takes an exclusion list. "Stop everything and restart the ambience" stops every sequence except the corridor alternation and the Donut Quiz spotlights:

```json
{ "command": "stop_all", "datas": { "exclude": ["lobby_prison_break_rest", "lobby_donut_quiz_scene"] } }
```

The Donut Quiz spotlights show what blinker-agent brings. When the brief starts, the 3 spotlights in front of the room switch on, then off by themselves 3 minutes later. We could have written a delayed handler of 180 seconds. But a two-step blinker sequence **can be cancelled**, which a delayed handler can't. If a GM turns the lights on manually, or another brief switches them off, the automatic switch-off is cleanly cancelled.

### Light launch, full launch

A request that came in along the way: for some rooms, the brief must **switch off the lights of every other room**, so that only the launched theme stays lit.

- **Light launch**: the room's sound and effects.
- **Full launch**: the same, followed by a `command_bundle_publish` of `lobby_<room>_lights_off` for every other equipped room. Each bundle first stops its room's sequence, then switches its outputs off.

```json
{
  "key": "lobby_prison_break_lights_off",
  "commands": [
    { "key": "lobby_prison_break_scene_stop",   "position": 1 },
    { "key": "lobby_prison_break_rest_stop",    "position": 2 },
    { "key": "lobby_prison_break_blue_led_off", "position": 3 },
    { "key": "lobby_prison_break_red_led_off",  "position": 4 }
  ]
}
```

A week later, the teams settled it: a single "Launch" button per room, "full" for rooms that have lights in the lobby, "light" for the others. Only the Haunted Room and Lost keep both. The handlers for both variants stayed in the gameflow, so bringing a button back only takes a data patch.

---

## The mini-game: the Haunted Room bell

This is the fun part. In front of the Haunted Room's façade, a haunted old hotel, sits a reception bell. A curious customer rings it:

1. the lobby ambience fades out in 3 s, and **one of 3 bell sounds** plays, picked at random;
2. when the bell ends, a **special Haunted Room ambience** starts, and the façade's windows and logo flicker faster and faster, then everything goes dark, then the logo alone pulses;
3. 10 seconds later, everything returns to normal.

The whole thing lasts 10 to 15 seconds, as the agency asked: the first version lasted as long as the special ambience, and the bell sounds were barely audible (we raised them to a −1 dBFS peak with a small `ffmpeg` script). The rules of the game are easy to state:

- ignored when the lobby is stopped;
- ignored **during a brief**: a customer must never be able to talk over the GM;
- ignored **during its own sequence**: ten rings don't restart it ten times.

### Hidden milestones as locks

Rather than inventing a notion of variables, we repurposed **milestones**. A milestone can be **hidden** in the GM UI, and validating or resetting one works outside a session: the state is stored, and only the session event fails, silently. We use them as locks:

| Hidden milestone | Validated by | Reset by |
|---|---|---|
| `lobby_running` | Start | Stop |
| `lobby_brief_running` | each brief | end of brief, "Stop the brief", "Stop everything", Start, Stop |
| `lobby_haunted_room_bell_running` | the bell | end of its sequence, a brief, Start, Stop |

The bell handler becomes a simple guard:

```json
{
  "key": "lobby_haunted_room_hotel_bell_pressed",
  "notification": "lobby_haunted_room_hotel_bell_pressed",
  "filters_behavior": "and",
  "filters": [
    { "behavior": "milestone_not_validated", "key": "lobby_brief_running",             "position": 1 },
    { "behavior": "milestone_not_validated", "key": "lobby_haunted_room_bell_running", "position": 2 },
    { "behavior": "milestone_validated",     "key": "lobby_running",                   "position": 3 }
  ],
  "behaviors": [
    { "behavior": "milestone_validate", "key": "lobby_haunted_room_bell_running", "position": 1 },
    { "behavior": "command_publish",    "key": "lobby_ambiance_fade_out",         "position": 2 },
    { "behavior": "command_publish",    "key": "lobby_haunted_room_bell_play",    "position": 3 }
  ]
}
```

The random pick needs no code. It's an existing command format, `audio_player_play_one_random`: the orchestrator picks the track and sends a regular `play` to the player.

```json
{
  "key": "lobby_haunted_room_bell_play",
  "recipients": [ { "type": "audio_player" } ],
  "format": "audio_player_play_one_random",
  "datas": [
    { "key": "tracks",         "value": "haunted_room_bell_1|haunted_room_bell_2|haunted_room_bell_3" },
    { "key": "volume",         "value": "1.0" },
    { "key": "at_end_message", "value": "lobby_haunted_room_bell_ended" }
  ]
}
```

The end of the sequence is a **delayed handler** (`"delay": 10`) on the same end-of-bell notification. One important detail: the orchestrator **re-evaluates its filters when it fires**. If a brief was launched in the meantime, it does nothing, and the brief keeps control.

### The last-element bug

While validating the random pick, we noticed the third sound never came up. .NET developers have probably guessed already. Simplified:

```csharp
var index = random.Next(0, tracks.Length - 1); // exclusive upper bound: the last item is never picked
```

`Random.Next(minValue, maxValue)` excludes `maxValue`. The fix was a one-liner, shipped in the messaging infrastructure package and then in maestro-room 1.16.3. We checked it on a local broker with the orchestrator's real *communicator*: 300 rings, 96 / 88 / 116 picks. The same bug affected `notify_one_random` and the `FirstRandom` helper, so other rooms too. Some gameflows even duplicated their last track to get it played at all. That duplicate now has to go, or the track comes up twice as often.

### The handler ordering trap

One last subtlety. When a brief starts during the bell sequence, a clean-up handler stops the scene and puts the façade back at rest (logo **on**). A **full launch**, however, must switch the façade off.

The orchestrator runs the handlers of a given notification one after the other, **single-notification handlers first**. The clean-up handler, which listens to fourteen notifications, would therefore run after the full launch and switch the logo back on right after the launch turned it off. The fix: the clean-up handler doesn't listen to full launches, which handle the bell themselves (stop the scene, switch off, release the lock). The rule is written down in the gameflow README, in the "Add a brief" checklist.

---

## Managing it all as data

### GM buttons are versioned patches

GM UI buttons don't live in the gameflow but in the database, imported from `_datas` files. Two rules we picked up reading the GM API's data layer:

- **Never edit the initialisation file** of an already initialised database. Everything goes through a versioned patch (`add`, `update`, `delete`), applied with `POST /datas/run`.
- **Never delete a button category.** Deleting a category also deletes the buttons it still contains, even if moving them out failed just before (a failed patch operation doesn't stop the next ones). So when we regrouped the buttons by room, the old "Mission briefs" category became… "Koh Lanta".

Switching to "one launch button per room" fits in a single patch: 11 buttons deleted (their actions and filters cascade) and 11 renamed.

### Eleven versions in two weeks

The gameflow went from 0.1.0 (27 October) to 0.11.0 (9 November). Each version has a dated entry in `_versions.json`, which also says **in which order to deploy it**: data patch without reset, then orchestrator restart, or the other way round. Updating the lobby means a push, then a "Restart room-orchestrator" button on the technical page. That pace let us iterate directly with the venue team: four versions on 3 November alone, including the shorter bell and the Prison Break corridor alternation, both requested that same day.

### Validating without the hardware

We wrote most of the gameflow before we had the server or the sounds. To avoid discovering mistakes on site, each building block was replayed against the **real code**:

- loading, through the real gameflow reader from `maestro-infrastructure`: no errors, every `at_end_message` has a handler, every button has a notification;
- the `_datas` files, deserialised with the GM API's import models;
- audio behaviour, replayed in the player's integration harness (real controllers, simulated mixer): end of a brief, a brief cutting another, a stop followed by another room's brief;
- light scenes, run through blinker-agent's `Sequence` class.

On site, only what no test can replace was left: volumes, swapped addresses, and how it all feels in the room.

---

## Operations: the `/tech` page

Every Maestro room has a technical page, also described in the gameflow. The lobby's page gathers:

- **status lights** (audio player, panels' broker);
- status, restart and logs for the audio player, io-status-agent and blinker-agent, over SSH;
- a **"Lobby sounds"** button that checks the 18 expected files and prints `OK` or `MISSING`. Four briefs were still waiting for their file at install time: a "Launch" button with no file lowers the ambience and plays nothing, and this check explains why straight away;
- a guarded **"Restart room-orchestrator"**: it first queries the room's state and refuses to restart during a state transition.

For GM instructions, we chose **one single safety button** over a list of procedures: "Stop everything and restart the ambience". It cuts everything, releases the locks, puts the façade at rest and restarts the ambience. It covers almost every degraded case: orchestrator restarted at the end of a brief, audio player restarted, a lock left behind.

One last operational point, valid for all our rooms: an I/O panel that (re)boots publishes its status and **ignores every command** until it gets an answer for each of its modules. Without `io-status-agent`, the lobby outputs stay dead, with no visible error anywhere.

---

## What's next

### Being tested in Lille: playlist, advertising and marketing TVs

The Lille lobby also serves as our test bench. Three features are being tested there, driven from Maestro like everything else:

- **background playlist management**;
- **advertising management**;
- the lobby's **marketing TVs**.

We'll come back to them in a future post, once they've proven themselves in real conditions.

### Microphone announcements

This one already exists: the GM UI can do *push-to-talk* to a room's audio player. Plugging a microphone into the front-desk station means a GM can call a team in the lobby, with zero development. We just need to offer it to the GMs.

### Paris 9, La Défense and Aéroville

Three more lobbies are being installed. The Lille patterns (briefs, locks, light sequences, full launch) serve as the template, but each lobby has its own electronics. Paris 9 sets the tone: its legacy controller's I/O map lists 17 inputs (8 of them virtual) and 56 outputs, against 18 I/Os in Lille. It includes per-room **start switches**, **pneumatic tubes** with per-theme messages, check-in spotlights, Fort Boyard's torches and hourglass, a fog machine, and "no customer" and "classic cycle" modes. The pneumatic tubes will deserve a post of their own.

---

## Takeaways

- **A generic platform delivers on its promise when you push it outside its use case.** A room with no game, permanent buttons and a bell mini-game took a single code change, narrow and tested.
- **Hidden milestones make excellent locks.** It's a repurposed feature, but it's readable, visible in the database, and resettable from a button.
- **Pick robustness over elegance.** An explicit fade rather than `notify`, a single file rather than chained tracks: each time, the option that doesn't depend on the orchestrator at a critical moment.
- **Write every decision down.** The scoping document numbers sixteen decisions, D1 to D16: settled, proposed or superseded. When a question comes back ("why not `notify`?"), the answer is written down, along with the trap that motivated it.
- **Two documents, two audiences.** A technical README for us (keys, files, installation) and a `scenarios.md` for operations, describing what happens event by event, without a single line of JSON. The second one is what the GMs actually read.

The Lille lobby is in service on Maestro, and Paris 9, La Défense and Aéroville are next. In the morning, a GM clicks **Start**. In the evening, **Stop**. In between, customers ring the haunted hotel's bell, and it still makes us smile.
