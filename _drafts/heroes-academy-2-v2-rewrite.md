---
title: "Heroes Academy (2/3): the V2 rewrite for Canada"
description: "No more Avalonia, no more MongoDB in the rooms, logic split from display and audio, hardware described as data: the architecture of Heroes Academy V2, and the bugs it did not magically fix."
lang: en
tags: [dotnet, csharp, mqtt, raspberry-pi, iot, architecture, lessons-learned]
---

# Heroes Academy (2/3): the V2 rewrite for Canada

> **The Heroes Academy series**
> 1. [Designing software for a physical game venue](./heroes-academy-1-v1.md)
> 2. **The V2 rewrite for Canada** (you are here)
> 3. [Eight months of AI-assisted development](./heroes-academy-3-ai-assisted-development.md)

In [part one](./heroes-academy-1-v1.md), I described Heroes Academy V1: one Avalonia application per Raspberry Pi, MongoDB used as an integration bus, a zombie process that forced us to reboot every room after each server restart, and hand-made deployments.

That V1 kept Lille and Tarbes running for years. But the opening of Brossard, near Montréal, was the trigger. Installing yet another copy of V1 more than 5,000 km away, with no way to drop by and reboot a Pi in the morning, was not an option. So I started a full rewrite in September 2025, and V2 was installed in Canada in early 2026, with three goals:

1. **More evolvable**: add a room, a game mode or a site without duplicating code.
2. **More stable**: no room may depend on a connection that cannot rebuild itself.
3. **Lighter to maintain**: no more Avalonia, no more manual deployments, and a single source of truth for configuration.

---

## The architecture

```text
┌──────────────────────────── Central server (Ubuntu 24.04) ───────────────────────┐
│  PostgreSQL ◄── WebApi (.NET 8, EF Core, MediatR)   SessionsOrchestrator (.NET 8) │
│                    ▲                                        ▲                     │
│  Admin UI / Kiosk UI (HTML + Web Components + mqtt.js) ─────┤                     │
│                    │ HTTP                                   │ MQTT                │
│                    │                       central Mosquitto┘                     │
└────────────────────┼────────────────────────────▲─────────────────────────────────┘
                     │                            │
┌────────────────────┼──────── Room Raspberry Pi (Debian, arm64) ──────────────────┐
│                    ▼                            ▼                                 │
│   RoomOrchestrator (.NET 8) ── game logic, one IGame per process                  │
│        │  JSON /ui            │ JSON /audio            │ /io/input, /io/output    │
│        ▼                      ▼                        ▼                          │
│   GameUI (Python, mpv)   MQTT audio player      local Mosquitto ◄─► I/O boards    │
│                                                        ▲                          │
│                                     ElectronicsBridge ─┘ (relay to central)       │
└───────────────────────────────────────────────────────────────────────────────────┘
```

On the central server:

- **PostgreSQL** replaces MongoDB;
- a .NET 8 **WebApi** (EF Core, Npgsql, MediatR) serves configuration and records sessions and scores;
- a **SessionsOrchestrator** moves teams from room to room.

In each room, a .NET 8 **RoomOrchestrator** runs the logic of a single game, chosen by configuration at startup. Two separate processes sit alongside it: **display** and **audio**.

One thing did not change: the electronics still speak MQTT with messages like `/c0m2p5/on`. It was V1's best decision, and I kept it as is.

---

## Six principles

### Principle 1: rooms never talk to the database

This is the direct lesson of the zombie process. A Pi knows only two things:

- **HTTP**, to fetch its settings at startup: durations, wiring, texts, videos. Settings are **cached locally**, with atomic writes and a backup copy, so a room can start even if the API is unreachable.
- **MQTT**, for everything that happens during the game, with a strict topic convention:

```text
/games/ha                          session bus (room ended, start the next one…)
/games/ha/rooms/{room}             commands to the room orchestrator
/games/ha/rooms/{room}/ui          screens
/games/ha/rooms/{room}/audio       audio
/games/ha/rooms/{room}/notifs      scores, persisted by the API
```

The server can restart: a well-written MQTT client reconnects. I'll explain further down why "well-written" is the hard part.

### Principle 2: separate logic, display and audio

In V1, a video-player crash took the game logic down with it. In V2, the screen is a **dumb terminal**. It receives JSON commands and displays them, knowing nothing about the rules:

```json
{ "command": "start_game",
  "room": "snake",
  "datas": { "lang": "en", "team": "The Invincibles", "players_count": 5, "duration": 450 } }

{ "command": "score_update",
  "datas": { "remaining_time": 212, "score": 340, "score_diff": 20, "is_penalty": false } }
```

The GameUI, which replaces Avalonia, is a lightweight Python application (Tkinter, with libmpv for video). It is packaged as a `.deb` and runs as a systemd *user* service under Wayland. It reuses Python modules (logging, MQTT, API client) already battle-tested on another of our products.

The timer runs locally to stay smooth, and the orchestrator **resynchronises** it at key moments: halftime, 60 s, 30 s, 10 s, and with every score update. Audio also lives in its own process, an MQTT-driven audio player with ducking and per-language tracks.

Each piece can restart independently of the others. And the game logic, in C#, became testable without a screen again.

### Principle 3: hardware is data

In V2, "button #3 of the Ropes room is on `c0m2p5`" is a **database row**, served by the API. It is no longer a JSON file copied onto each Pi. The same binary runs on every site, and physical differences are described in data.

The Brossard Snake shows why this matters. In France, the grid is an 8 × 8 floor wired to I/O boards. In Brossard, it is an **8 × 10 grid** of tiles from another manufacturer, connected over **USB/FTDI**, with a dedicated agent publishing `{x, y, state}` on MQTT. On the game side, all it took was an adapter and a grid size made configurable per site.

### Principle 4: a game is a step machine

Every game derives from a `BaseGame` that runs a step machine. It covers the room start, audio instructions, game start, halftime, siren, countdown, bonus question, answer reveal and results. All of it is driven by a single timer, guarded by a generation counter so that a room that has ended cannot restart its own timer.

Most of the behaviour is shared, so a "simple" game room fits in a few lines:

```csharp
// Ropes: climb and hit the 2 red targets that light up at the same time.
public sealed class RopesGame : BasePairSequentialGame<GenericCustomSettingsApiDto>
{
    public RopesGame(IRoomSettingsService settingsService, ISessionDataService sessionDataService,
        ISecondaryMissionGame secondaryMissionGame, IMqttPublisherFactory publisherFactory,
        ILogger<RopesGame> logger, IGameCustomSettingsHttpDataService customSettingsHttpDataService)
        : base(RoomsEnum.Ropes, settingsService, sessionDataService, secondaryMissionGame,
               publisherFactory, logger, customSettingsHttpDataService)
    { }
}
```

Richer rooms such as Balls or Snake only override what makes them different.

### Principle 5: everything must run without the hardware

I couldn't debug Canada from France with a screwdriver. So I had to be able to reproduce the site without a single physical button.

The first tool, there from the start of the rewrite, is **a whole room in Docker**. The orchestrator, GameUI and audio player run in one container, the screen is exposed over noVNC and the audio is streamed to the browser through a PulseAudio sink. Audio in a container held out longer than anything else, until we switched from PipeWire to plain PulseAudio. And since the electronics speak MQTT, a room can then be driven with `mosquitto_pub`.

Other tools came after the opening:

- a **browser-based test UI**: it draws the room from the API (buttons, LEDs, tiles, doors) and every click publishes to `/io/input`, exactly as a board would;
- a command-line **robot player**, with tunable error rate and reaction time, for the most complex games;
- above all, an **automated test suite**. Time is an injected dependency (`TimeProvider`), so a 7-minute game is tested in a few milliseconds. It had about ten tests when Canada opened and has over 800 today. I come back to that evolution in [part three](./heroes-academy-3-ai-assisted-development.md).

### Principle 6: deployment is a feature

Brossard was installed using documented room-by-room procedures: `.deb` packages, systemd services, one configuration per machine. That was already a clear improvement over V1's zip files, but it was still long and error-prone.

Since summer 2026, a **one-command deployment tool** has taken over. An inventory file describes the site, one line per machine:

```text
"Snake|pi-snake|snake|gameui,audio,iostatus,orchestrator,bridge,mosquitto|"
```

The rest follows:

```bash
./deploy.sh -i inventory/<site>.env preflight            # SSH, sudo, architecture
./deploy.sh -i inventory/<site>.env --dry-run install    # dry run
./deploy.sh -i inventory/<site>.env install              # server, then rooms in parallel
./deploy.sh -i inventory/<site>.env --limit snake update # a single room
./deploy.sh -i inventory/<site>.env verify               # status of every service
```

.NET binaries are published self-contained, so no runtime needs to be installed on the Pis. All configuration is generated from templates, and nothing is edited by hand on the machines. One constraint remains: the machines sit on a private LAN, so deployment is launched from a computer plugged into the site's network.

---

## One codebase, two game modes: migrating Lille

Brossard has no Arena: the bonus question is asked at the end of each room. Lille, on the other hand, wanted to keep its Arena. When Lille migrated from V1 to V2 in September 2026, V2 therefore had to support both modes: a question at the end of each room, or a 45-second visit to the Arena (plus a safety margin) between two game rooms.

A site never changes mode. So I chose to make it a **compile-time constant** rather than a database setting: `dotnet publish -p:GameMode=Arena`. Artefacts are stamped with their mode, and the deployment tool refuses to mix binaries from two modes on the same site. A configuration that cannot be changed by mistake is one you don't need to monitor.

The Arena also brought back an old V1 problem: two teams must never meet there. In V2, this became a visit identity (`RoomSessionId`) carried end to end, from scheduling to MQTT messages, plus regression tests on crossings.

Tarbes still runs V1, kept stable by the watchdog described in part one.

---

## Operating from afar

For a site six hours away, the admin has to make it possible to act without being on site:

- restart a Pi or a service from the UI, over SSH;
- force-end a stuck session, with every Game Master action logged;
- view the cameras;
- talk into a room with push-to-talk: the PCM stream goes over WebSocket to the room's audio player, one speaker at a time.

For statistics, a separate analysis tool reads PostgreSQL dumps and produces KPIs and charts. For example, it flags sessions that were force-ended.

The admin UI is deliberately plain: static HTML, native Web Components and mqtt.js over WebSocket, with no build step.

---

## What V2 did not magically fix

A rewrite doesn't remove bugs, it changes their nature. Here are the ones that taught me the most.

### Reconnected, but deaf

This is the most instructive story of V2. On the Canadian site, the operator suffered regular MQTT disconnections that made the room unplayable. The `SessionsOrchestrator` did reconnect to the broker. It simply **heard nothing after reconnecting**.

Digging in, I found a set of defects that reinforced each other:

1. **No resubscription.** The subscription to `/games/ha` was made once, at startup. But each reconnection opened a brand-new broker session (*clean session*): the client came back connected, with no subscriptions at all.
2. **A ClientId regenerated on every attempt** (`ClientId + "-" + Guid.NewGuid()`), which piled up ghost sessions on the broker.
3. **Re-entrancy.** MQTTnet raises `DisconnectedAsync` even when a `ConnectAsync` fails. The disconnection handler started a new connection, which waited on a lock held by the attempt that was failing.
4. **A typo.** The code read `"Mqtt:Post"` instead of `"Mqtt:Port"`, so the configured port was ignored.
5. **No tests** for this service.

The fix was specified before being coded, with about twenty acceptance criteria. The core principle is **a single reconnection authority**. The disconnection handler only wakes up a single loop, which reconnects, resubscribes to everything and waits for the next loss:

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    // The signal is armed BEFORE the attempt: a disconnection reported
    // during or right after connecting cannot be lost.
    var signal = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
    Volatile.Write(ref _disconnectedSignal, signal);

    if (await TryConnectAsync(port, stoppingToken))
    {
        delay = _minDelay;
        await ResubscribeAllAsync(stoppingToken);          // on EVERY connection
        await signal.Task.WaitAsync(stoppingToken);        // wait for the next loss
    }

    await _delayProvider.DelayAsync(ApplyJitter(delay), stoppingToken); // 1 s → 30 s, with jitter
    delay = IncreaseDelay(delay);
}
```

The expected behaviour is:

- bounded exponential backoff, with jitter so that every service doesn't rush the broker at the same instant;
- a stable ClientId for the whole life of the process;
- delays abstracted behind an interface, so that tests are deterministic.

The deliverable was built on a base aligned with the production version, not on `main`, so that it shipped this fix and nothing else.

### The screen that froze

On the GameUI side, a screen sometimes froze mid-game, with no further score updates. The cause came down to a single call.

The Python MQTT library (paho) calls `on_message` on **its own network thread**, the very thread that must send the keep-alive PINGREQ. But we were building the next screen there, and loading a background image alone takes about a second. Meanwhile the keep-alive stopped going out, the broker considered the client dead and disconnected it (`rc: 7`).

The fix is to hand the work over to the Tk thread:

```python
def _show_screen_on_ui(self, screen_id, data=None):
    # Never do heavy work on paho's network thread: hand it to the Tk thread,
    # which is the only thread allowed to touch widgets anyway.
    self.ui.root.after(0, lambda: self.ui.show_screen(screen_id, data))
```

It's a rule every UI developer knows: never touch widgets outside the UI thread. Here it comes with a networking rule: never block the thread that keeps the connection alive.

### 8 pm in Montréal

A bug report from Canada: "sessions disappear from the dashboard every evening around 8 pm". The query filtered sessions on the **UTC calendar day**. In Montréal, on summer time, UTC−4, that day rolls over at 8 pm local time.

The fix replaced the calendar day with a rolling 24-hour window. But the real lesson is that software designed in France had never had to ask itself *what day it is*. Since then, all the analysis tools work explicitly in `America/Toronto`.

### Concurrency, hardware and other delights

- **An ignored answer.** An answer to the bonus question arriving during the reveal window was lost because of a state lock, since replaced with `Interlocked` operations.
- **The Balls baskets**, scored concurrently, needed one lock per basket.
- **A hang in the Arena.** A team arriving while the previous team was finishing the room overwrote a shared session identifier, and the session got stuck.
- **"Ghost" sessions** that never started pushed the calculation of the following slots into the past. We had to exclude them explicitly, then add detection of stalled sessions and a clean forced end.
- **Spurious pulses** on the Don't Touch The Floor floor sensors called for a configurable debounce. On the magnet panels of the Communication room, transitions less than 200 ms apart are ignored.
- **Starting under Wayland.** The GameUI sometimes started before the compositor. The service now waits for the Wayland socket before launching.

---

## What's next: The Switchboard

With the platform stable, I could get back to what matters to players: the game. The latest project is **Le Standard** ("The Switchboard"), a new gameplay for the Communication room. As I write these lines, it is being tested on site, a few days away from going live.

One player becomes the operator and reads calls on a console, and the others must patch them through on magnet panels. The game engine is **pure**: no MQTT, injected clock. Every phase (rewiring, switchboard, shift change, overload, shutdown) can therefore be tested like a function.

This project also reminded me that we design for humans. Players have learned to **leave the room as soon as a beacon lights up**. So the Switchboard uses none during play: everything goes through sound and screens. And the end-of-game "general shutdown", which drops the magnet panels after a "Step back! 3-2-1", is disabled by default. It is only enabled once safety has been validated in the room.

---

## What V2 taught me

1. **Separate what crashes from what matters.** Display and audio can restart; the game logic must keep going.
2. **Hardware is data.** Wiring in code means one binary per site.
3. **Time is a dependency.** For tests (`TimeProvider`) as much as for production (time zones, timer resynchronisation).
4. **Reconnection must be tested.** "It reconnects" means nothing until you've checked that it listens again.
5. **Deployment is part of the product.** One command, a dry run, a verification step: that's what makes it possible to run a site across the Atlantic.
6. **We design for humans.** Players who flee beacons, a Game Master who must restart a room without a terminal: the architecture has to serve these people.

---

## The players' verdict

After months of work, late-night deployments and remote debugging, V2 delivers: the rooms run, operations are calm, and above all, players come out delighted. I'll let them have the last word.

<!-- Google review screenshots go here -->

![Google review](./images/google-review-1.png)

![Google review](./images/google-review-2.png)

---

> **Next in the series**
> 3. [Eight months of AI-assisted development](./heroes-academy-3-ai-assisted-development.md): from March 2026, AI took a growing share of V2's development. What the Git history says, with numbers.
>
> To (re)read the beginning: [1. Designing software for a physical game venue](./heroes-academy-1-v1.md).

*Code excerpts in this article are simplified. Addresses, credentials and site-specific infrastructure details have been deliberately left out.*
