---
title: "Donut Quiz: moving a game from .NET 5 and Avalonia to the web, with the rooms still open"
date: 2026-11-12
lang: en
tags: [donutquiz, dotnet, avalonia, sse, mqtt, raspberry-pi, dmx, gpio, debian, migration]
---

> **Alternative titles** (pick one, then delete this block)
>
> 1. Six desks, one buzzer, no host: behind the scenes of Donut Quiz
> 2. From Avalonia to EventSource: moving Donut Quiz to the web
> 3. One quiz, 9 sites, 81 machines: Donut Quiz's architecture and lessons

# Donut Quiz: moving a game from .NET 5 and Avalonia to the web, with the rooms still open

In the previous articles I talked a lot about escape rooms. But the product that occupies the most machines in our fleet isn't one: it's **Donut Quiz**, a team quiz installed at our sites, which on its own accounts for nearly half of the machines [we operate](./vigie-exploitation.en.md).

This article covers its architecture, its hardware quirks (buzzers, DMX, sound), and above all the migration we carried out in 2026: moving off a .NET 5 and Avalonia stack on 32-bit Raspbian to .NET 10, web views in vanilla JavaScript pushed over SSE, and Debian 13 on arm64. All without closing the rooms.

## The game

A game pits 2 to 6 teams against each other. Each one sits at a coloured **desk** named after a donut flavour: Chocolat, Citron, Myrtille, Pistache, Fraise, Caramel. A desk is:

- a **touch screen** for questions and answers;
- a **score screen**, facing the room;
- a physical **buzzer**;
- an RGB **LED bar**.

A large shared screen, the *saloon*, shows the game to everyone, and DMX spotlights light each team. There's **no host**: voice-over, slides and videos run the show. The game master only creates, starts, pauses or stops a game.

The rounds have food names: *Pour du beurre* (one warm-up question), *Mise en bouche* and *Surprise du chef* (15 questions each), two 30-second mini-games (*Apéro* and *Milkshake de frites*), then *Cerise sur le gâteau* (10 questions of 45 s each). A normal game has 40 questions and lasts about an hour; a short mode (15 questions, about twenty minutes) and an express mode round out the offer.

Teams have **special strikes**: *Bouchées doubles* (double points), *On fait moit-moit* (half the wrong answers disappear), *Pique-assiette* (steal half of a team's points on the question), *Retard en cuisine* (the targeted team loses 9 seconds of display time). At the end, awards are handed out: the *Donut d'or* for the winning team, the *Steak saignant* for the team that got robbed the most.

On the mechanics side, one detail matters a lot: **you touch the screen to choose, you press the buzzer to commit**. Response time is measured on the desk, and correct answers are ranked by speed.

## Nine Raspberry Pis per site

Donut Quiz runs on 9 sites, with 9 machines per site, mostly Raspberry Pi 4s (8 GB):

```
                       ┌──────────────────────────────┐
                       │ Central server (Pi 4)        │
                       │ admin · check-in · sync      │
                       │ MongoDB · Mosquitto          │
                       └──────────────┬───────────────┘
                                      │ MQTT  /games/dq/…
          ┌───────────────────────────┼─────────────────────────────┐
          │                           │                             │
 "donut" (Pi 4)                 6 desks (Pi 4)                 saloon (Pi 4)
 game engine                    touch + score                  big screen
 sound (MQTTaudioplayer)        GPIO buzzer                    (3 TVs via a splitter
 spotlights (DMX)               LEDs (DMX)                      at one site)
```

Everything talks MQTT, under a single prefix:

```csharp
private const string GAME_TOPIC = "/games/dq/";
public static string GetOrchestratorTopic() => string.Concat(GAME_TOPIC, "orchestrator");
public static string GetSpotlightsMasterTopic() => string.Concat(GAME_TOPIC, "lights");
public static string GetSoundMasterTopic() => string.Concat(GAME_TOPIC, "audio");
public static string GetDesksTopic() => string.Concat(GAME_TOPIC, "desks/all");
public static string GetDeskLedColorTopic(PupitreColors color) => string.Concat(GetDeskTopic(color), "/led");
```

It wasn't always so: the first version used Akka.NET, and commented-out traces of it remain in the code. Moving to MQTT aligned Donut Quiz with the rest of our products, and that choice is what made the migration described below possible.

A game follows a simple path:

1. the game master starts the game from the admin, which publishes a `StartGame` command on `/games/dq/orchestrator`;
2. the game engine, on the "donut", fetches the full game over HTTP;
3. it preloads every sound of the game into the audio player;
4. a *storyboard* plays the steps, and each step publishes to the saloon, the desks, the sound, the spotlights and the LEDs;
5. the desks answer on the orchestrator topic (team registration, answer, special strike…), and the server sends them a delivery receipt;
6. rankings, events and awards go back up to the admin.

## The engine: a storyboard written as an iterator

The heart of the engine is an elegant choice: a game's flow is a **C# iterator**. Each `yield return` produces a "story" (a step, a conditional loop, a technical action), and a one-shot timer moves from one step to the next according to each step's duration.

```csharp
yield return new WhileStory(GameRounds.Game,
    () => new TeamsSettingUpInProgressGameStep(TEAMS_SETTING_UP_CHECK_INTERVAL_GAME_STEP_DURATION, ...),
    () => !IsTeamsSettingUpFinalized());
yield return new IdleStory(GameRounds.Game, () => new TeamsSettingUpResultGameStep(...));
yield return new TechnicalStory(GameRounds.Game, () => this.ProvideTeamsToAskers());
...
foreach (var round in Rounds.Where(round => !round.GameRound.In(GameRounds.PourDuBeurre)))
{
    yield return new IdleStory(current.GameRound, () => { this.currentGameRound = current; return current.Start(TeamManager); });
    foreach (var story in round.Stories) yield return story;
    yield return new TechnicalStory(current.GameRound, () => { this.teamManager = current.End(); });
}
```

The flow reads top to bottom like a show script: wait for teams to register, announce the teams, then play each round. The timer driving the iterator carries a comment I like, because it owns up to bending best practice: *"async void is intentional here: System.Timers.Timer.Elapsed requires a void return type. All code paths are wrapped in try-catch"*.

Rounds fall into two families. **Server rounds** (the multiple-choice questions) are driven message by message by the engine. **Client rounds** (the 30-second mini-games) are sent to the desks in one block; the desks play them on their own and send back the result. For 12 questions in 30 seconds, a network round trip per question would make no sense.

## The buzzer, the game's only input

On a desk, the buzzer is a button wired to a GPIO pin on the Raspberry Pi. In the first version, the desk application read it directly:

```csharp
controller ??= new GpioController(PinNumberingScheme.Board);
controller.SetPinMode(BUTTON_GPIO, PinMode.InputPullDown);
controller.RegisterCallbackForPinValueChangedEvent(BUTTON_GPIO, PinEventTypes.Rising, OnPinValueChanged);
...
Monitor.TryEnter(padlock, TimeSpan.Zero, ref lockTaken);
if (lockTaken && DateTime.Now.Subtract(last).TotalMilliseconds > 500)
{ BuzzerBumped?.Invoke(this, EventArgs.Empty); last = DateTime.Now; }
```

Pin 40 (BCM 21), pull-down, rising edge, and a 500 ms debounce under a non-blocking lock.

In the new version, the buzzer became a small standalone Python service that publishes a message on `/games/dq/desks/<colour>/buzz`. Its header tells the story of one of the most frustrating days of the migration. At Paris 9, on 19 August, four desks out of six stopped responding to the buzzer. Yet the service log showed every press. The first defect is described like this (translated from the comment):

> The service logged "buzz #N" right after client.publish(), which is ASYNCHRONOUS. On four desks out of six, the log showed presses while the broker had received NOTHING for hours. The service was lying, and the diagnosis went off the rails (wiring, GPIO, buttons) for an entire day.

Three fixes came out of it:

- **tell a detected press from a press confirmed** by the broker (`on_publish` callback), and count lost presses: *"An MQTT client that doesn't lie: it distinguishes sent, confirmed and lost"*;
- **connect asynchronously and retry forever**, including on the very first attempt: after a power cut, a desk often boots faster than the server;
- **watch the MQTT client's network loop**: a client can stay "connected" at the socket level with a dead loop. After 60 seconds without a connection, the service exits on purpose and systemd restarts it.

```python
# Beyond this delay without a broker connection, we kill ourselves: systemd restarts a clean
# process. Better a 3 s restart than a live service that no longer emits anything.
MAX_DISCONNECTED_S = 60
WATCHDOG_PERIOD_S = 5
```

Even the logs get their own comment: every line is written with `flush=True`, because otherwise lines stay in the buffer and only show up when the service dies, "useless for diagnosing a room in the middle of a game".

## Light: hand-written DMX

Each site has six DMX spotlights, one per team: spotlight *n* lights desk *n*. The first version drove an FTDI FT232 USB adapter through the D2XX driver, generating the DMX frame by hand:

```csharp
ftdi.SetBreak(true);  Thread.Sleep(1);   // ~1ms break (spec min is 88µs)
ftdi.SetBreak(false); Thread.Sleep(1);   // MAB
var buffer = new byte[1].Concat(data).ToArray();   // start code 0x00 + channels
ftdi.Write(buffer, buffer.Length, ref written);
```

Each desk also has its own DMX controller for its LED bar: pink while waiting, green for a right answer, red for a wrong one.

Moving to our generic MQTTdmx agent brought two lessons. The first is a classic: the old code addressed channels from 0, while the new agent uses absolute channels from 1 and ignores channel 0. At one site, the log counted 152 "Command ignored, channel: 0" lines before someone changed a global offset from 0 to 1. The second: the old controller only wrote on each command, while the new agent streamed continuously, and spotlights set to HOLD mode started flickering. Hence the `send_on_change` mode described in the [first article](./team-break-architecture.en.md).

## Sound: a proper investigation

A site has between 3,300 and 3,600 sounds: voice-overs, ambiences, jingles, answers. They're played by MQTTaudioplayer, our generic audio player, as 44.1 kHz 16-bit WAV.

Overlapping sounds and unexplained silences led to a real investigation, written up in an analysis document: numbered causes, likelihoods, ways to tell them apart, fix status. On the engine side, four causes were identified and fixed. The most instructive is a `return` instead of a `continue`: a delayed audio instruction aborted every following instruction of the step, and the task carrying it couldn't be cancelled when the game stopped.

```csharp
if (audioInstruction.Offset > 0)
{
    // Delayed instructions must not abort the remaining instructions of the step,
    // and must be cancellable when the game stops ...
    CancellationToken delayedToken = _delayedAudioInstructionsCts.Token;
    _ = Task.Run(async () => {
        try { await Task.Delay(audioInstruction.Offset, delayedToken);
              await RunAudioInstructionsAsync(audioInstruction, delayedToken); }
        catch (OperationCanceledException) { }
        catch (Exception ex) { _logger.LogError(ex, "..."); }
    }, CancellationToken.None);
    continue;
}
```

Audio publishes also moved from QoS 0 to QoS 1. And "smoothing" was added: during the answer phase (the "tick-tock"), only the round's ambient loop is faded to zero, through a new `FadeTrack` command, instead of cutting everything.

An audit of the sites revealed another, more mundane cause of silence: between 149 and 384 sounds were missing per site, about 1,450 in total, because the audio conversion job wrote its files outside the mounted volume.

## Content: questions, themes, and an editor with AI

A question has one right answer and 2 to 11 wrong ones, an optional "joke answer", a tip, a voice-over line, a visual (image or video), a sound for the right answer and an explanation. It belongs to a theme, a category and tags. **Themed games** ("100% Music", "100% Horror", a kids' version) are built from those tags. When a game is built, the history of questions already seen by customers is excluded: a returning team doesn't get the same questions again.

Content is prepared with a **dedicated editor**, a Python desktop application (PySide6) that edits the question file and tracks each question's origin (legacy, modified, new). A scanner flags missing media before export. An AI panel, wired to Claude or Gemini, generates questions and **audits** others, with a mandatory strict-JSON answer and explicit flags: `outdated`, `ambiguous`, `wrong_answer`, `too_easy`, `too_hard`, `not_so_fun`. The AI-produced columns are stripped from the export sent to the game. The editor ships as a Windows executable through a CI that tags the version and publishes the release.

### Syncing head office and sites

Content is created at head office, then **synchronized** to each site by Hangfire jobs:

- content (themes, themed games, questions, slides, texts, awards) is pulled every night, at a random time between 10 p.m. and 3 a.m., with an offset per job, so sites don't all hit head office at once;
- games and customers sync every 5 minutes, and played games go back to head office every hour.

Sync is incremental, with a watermark that **only advances if every item went through**:

```csharp
DateTime? since = contentSynchronizationInfo.Since;
var pulled = await reader.PullAsync(since);
foreach (var current in pulled.EmptyIfNull())
{
    try { await writer.PushAsync(current); }
    catch (Exception ex) { hasErrorOccurred = true; this.logger.LogError(ex, "..."); }
}
if (!hasErrorOccurred) { contentSynchronizationInfo.Since = executionDate; }
```

A single failing item replays everything on the next pass: more expensive, but nothing is ever skipped. Our maintenance procedure adds a common-sense rule: **a green job proves nothing**. Check the local database, and check that a media file actually answers HTTP 200. Media (8 to 9 GB per site) follow their own path.

## The migration: leaving Avalonia without stopping the rooms

### Where we started

At the start of 2026, Donut Quiz ran on a stack that had aged:

- **.NET 5**, long out of support;
- desk and saloon applications in **Avalonia 0.10.5**, with LibVLC for video and an embedded GIF decoder;
- Raspberry Pis on **32-bit Raspbian Buster** (armhf), on X11;
- deployment via **zip archives** unpacked by hand on each machine, since the original pipelines were no longer available.

The migration was run with a log of 16 ADRs. Here are the ones that shaped it most.

### The discovery that changed everything (ADR-0003)

Moving to .NET 10 meant migrating Avalonia from 0.10 to 11, a major break. Re-reading the code, one discovery simplified everything (translated):

> the desks are already **thin MQTT clients** — the `GameServer` broadcasts JSON DTOs (`DeskMessage`) describing what to display; the Avalonia ViewModels are nothing more than a "message → screen" state machine.

Hence the decision: replace the Avalonia views with **static web views in vanilla HTML, CSS and JavaScript**, served by a new ASP.NET Core host, with state pushed over **SSE** (`EventSource`) and inputs sent with `fetch()`. Blazor was ruled out: a framework, plus a permanent WebSocket or a WebAssembly runtime, was too heavy for the minimalism we wanted.

### A bridge, and zero changes to the engine (ADR-0005)

The new host subscribes to the desk and saloon MQTT topics, keeps the last message for each screen and pushes it over SSE (`GET /desk/{colour}/stream`). A screen that connects immediately gets the latest state. In the other direction, `POST /desk/{colour}/answer` publishes the message the engine expects on the orchestrator topic. The result: **no change to the game engine or to the contracts**, validated end to end.

Each desk now shows two kiosk browsers on two URLs (the touch screen and the score screen), subscribed to the same stream.

### Reproduce before improving (ADR-0006)

The most important decision isn't technical. During the port, some behaviours got *invented*: the buzzer mapped to the space bar, the answer sent on touch. They were wrong, and costly to detect. Hence a top-priority rule: **reproduce the existing behaviour**, reading the old application's code before writing the new one. "Invent nothing." Touch selects, the buzzer commits, as before.

The Avalonia projects were first frozen out of the build, then deleted from the repository at cutover (ADR-0010). No big bang.

### Changing CPU architecture (ADR-0014 and 0016)

A `dist-upgrade` doesn't change the architecture: an armhf Pi stays armhf. But the target (.NET 10, Docker, MongoDB) is arm64. So we had to **reflash**, and do it reproducibly:

- **golden images per role** (desk, saloon, donut, server), generated with `rpi-image-gen` on a minimal Debian 13 base, arm64 and Wayland (labwc), with every version pinned;
- a **`donutquiz.conf` file on the FAT boot partition**, editable from any PC without booting the Pi: `ROLE`, `COLOR`, `SERVER_URL`… On first boot, a *oneshot* service reads the file, generates the configurations, recalibrates the touch screen, sets the hostname, then enables the services.

The field procedure fits in three words: flash, edit `donutquiz.conf`, plug in. An SD-card replacement guide lets a site manager get a desk back into service in about twenty minutes.

### Packages that install nothing active (ADR-0012)

Each component became a **versioned Debian package** (`donutquiz-ui`, `-buzzer`, `-screens`, `-gameserver`, `-admin`…) that **activates nothing** on install. Cutover is explicit and reversible thanks to three tools: `dq-cutover` to switch over, `dq-rollback` to go back, `dq-doctor` to check health. The old and new stacks live side by side on the same machine (separate database, settings and media): rolling back means stopping one service and starting another.

### Rolling out, site by site

Aéroville went first, in early August, while the room kept playing. Then La Défense and Paris 9, then Lille. Each site brought its own lesson:

- two Docker Compose projects shared a name, and `--remove-orphans` stopped the neighbour's services;
- ASP.NET's data protection keys lived inside the container: every rebuild silently broke the admin buttons;
- a dependency declared as `Recommends: mongodb-clients` upgraded MongoDB without warning, from 4.4.8 to 4.4.29;
- and **MongoDB 4.4.19 and later require ARMv8.2-A**, which the Raspberry Pi 4's Cortex-A72 doesn't have. Symptom: `Illegal instruction`. Remedy: pin `mongo:4.4.18` and `apt-mark hold`;
- reflashed desks kept the default hostname and shared the same SSH host keys.

Today our four company-owned sites run on the new stack; the franchises are following.

## The desks that went deaf

Moving to the web introduced a new failure mode. In Lille, on 11 September, **five desks out of six had been deaf for hours, without a single error anywhere**. The transport module's comment explains why (translated):

> on a half-open TCP connection (switch, access point, NAT table), the browser waits for bytes that will never come, readyState stays OPEN, no onerror fires, and EventSource's native reconnection never triggers.

The fix covers both ways a stream can die: closing (`onerror` with `readyState === CLOSED`) and **prolonged silence**, detected by a liveness timer. A subtlety: an SSE comment (`: ping`) doesn't trigger `onmessage`, so the server's keepalive was invisible on the client. The server now also emits a named `ping` event, which the client listens to explicitly.

```js
/** Tolerated silence (no bytes at all: no message, no keepalive) before reopening the stream. */
export const LIVENESS_TIMEOUT_MS = 45000;

/** First reconnection step. */
export const RECONNECT_MIN_DELAY_MS = 1000;

/** Reconnection cap: we slow down, we never give up. */
export const RECONNECT_MAX_DELAY_MS = 30000;
```

The thresholds are chosen, not suffered: 45 s is three missed keepalives, and reconnection caps at 30 s so that a room with thirteen streams doesn't turn a briefly slow server into a reconnection storm. A ping-pong round trip also reports each screen's liveness all the way up to the admin.

## Archipel and vigie

At sites equipped with [Archipel](./archipel-orchestration.en.md), the front-desk kiosk also drives Donut Quiz. There's no Maestro in a Donut Quiz room, so no MQTT between the two: Archipel calls Donut Quiz's API over HTTP. It finds the game from the booking, passes on the theme chosen by the players, the number of teams and the language, then starts the game. The kiosk never offers more teams than there are working desks.

The HTTP client on the Archipel side never throws. Each call has a 5-second timeout, within an 8-second overall budget, and the comment gives the reason: "a Donut Quiz room that's switched off must not freeze the players' screen."

On the operations side, [vigie](./vigie-exploitation.en.md) watches the services of every Donut Quiz machine, named by function (`donutquiz-<site>-centralserver`, `-gameserver`, `-desk<colour>`, `-saloon`). It was a Donut Quiz sync service that had restarted thousands of times without anyone noticing, which put restart counters on vigie's roadmap.

## By the numbers

| | Old stack | New stack |
|---|---|---|
| Runtime | .NET 5 | .NET 10 |
| Screens | Avalonia 0.10.5 | vanilla HTML/CSS/JS + SSE |
| Pi OS | Raspbian Buster armhf, X11 | Debian 13 arm64, Wayland |
| Deployment | zip by hand | Debian packages, cutover/rollback |
| Tests | 88 test methods | nearly 2,000 |

The old solution had 38 projects and nearly 99,000 lines of C#.

## What I take away from it

1. **A thin client is a gift.** Because the desks only displayed MQTT messages, we could replace the whole interface without touching the game engine.
2. **Reproduce before improving.** In a migration, every behaviour "improved" along the way is a bug for somebody. Read the old code first.
3. **A service that lies is worse than a dead one.** The buzzer that logged presses it never sent cost us a day. A log must say what *happened*, not what was *requested*.
4. **An open connection isn't a live connection.** Half-open TCP, EventSource stuck in `OPEN`: only an active liveness check catches it.
5. **Migrate with the room open, side by side, reversibly.** Both stacks coexist, and rolling back is one command.
6. **Hardware makes its own rules.** A dist-upgrade doesn't change the architecture, and a minor MongoDB version can require an instruction your CPU doesn't have.
7. **Content is a product in its own right.** Editor, assisted audit, per-customer history, sync: the best architecture won't save a wrong question.

When six teams fight for the *Donut d'or* and every buzzer clicks in the same second, nobody thinks about the 81 machines, SSE or DMX. That's exactly the point.
