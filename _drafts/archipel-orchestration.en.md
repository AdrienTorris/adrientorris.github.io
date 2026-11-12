---
title: "Archipel: orchestrating the orchestrators"
date: 2026-11-12
lang: en
tags: [archipel, maestro, orchestration, mqtt, sse, dotnet, escape-room, architecture]
---

> **Alternative titles** (pick one, then delete this block)
>
> 1. Ask, then observe: how Archipel drives a site's Maestro instances
> 2. The stage manager and the conductor: Archipel above Maestro
> 3. From booking to countdown: orchestrating an escape-room site with Archipel

# Archipel: orchestrating the orchestrators

In the [previous article](./maestro-orchestration.en.md) I described how Maestro orchestrates an escape room: notifications come in, commands go out, milestones pile up, a countdown runs. But a room doesn't live on its own. At Aéroville, one of our sites, about ten rooms run side by side: up to nine Maestro instances, plus a Donut Quiz that doesn't use Maestro at all.

So there has to be a level above, one that answers questions no single room can settle:

- which team plays where, at what time, in which language, with how many players?
- when can a room's tablet be opened to players?
- when should the start order be sent, and to which instance?
- which variant of the room (the scenario) should be prepared for the next session?
- what should the lobby TVs show?

That's the job of **Archipel**, our site operations platform. If Maestro is the conductor of a room, Archipel is the theatre's stage manager: it doesn't play and it doesn't conduct, but nothing starts without it.

## Two levels of orchestration, one clear boundary

The most important design point fits in one table: **who is authoritative for what**.

| Archipel is authoritative for… | Maestro is authoritative for… |
|---|---|
| Bookings, slots, sessions | The actual state of the room |
| The team: player count, language, name | Time: start, duration, remaining time |
| The scenario of the *next* session | The gameflow and its execution |
| Launch words, access to the tablets | Hardware, agents, safety |
| What the lobby screens show | Milestones, hints, the end of the game |

Three rules follow from that split, and they show up everywhere in the code:

1. **Archipel asks, Maestro confirms.** Archipel never writes the state of a Maestro-driven room itself: it sends a request, then waits for the confirmation.
2. **To Maestro, Archipel is an optional observer.** The room's local MQTT broker is critical; Archipel's isn't. A room must be able to play, get prepared and stop while Archipel is down, under maintenance or on another subnet. On the Maestro side, the connection to Archipel never blocks startup, and publishing to Archipel is best-effort: "a lost notification must not turn an already persisted state change into a 500."
3. **Tablets only talk to Archipel, never to the game system.** The comment in the check-in endpoints says so explicitly: a future system (Heroes Academy) "will plug in here with no new path on the UI side." Archipel already drives two systems today: Maestro over MQTT, and Donut Quiz over HTTP.

Technically, Archipel is a .NET 10 application (EF Core and PostgreSQL 16, MQTTnet 5) running on one mini PC per site, in Docker Compose. Application code follows a command/query split with a minimal home-made mediator, and the front end is vanilla JavaScript (ES modules, no npm, no bundler). Several background services keep the site ticking:

| Service | Cadence | Role |
|---|---|---|
| Room sync | 24 h | Opens a "gate" that the other syncs wait on |
| Booking sync | 5 min | Imports bookings from the online booking tool |
| Slot sync | 1 h, 14 days ahead | Imports slots |
| Waiting-for-players switch | 30 s | Opens ready rooms whose slot is close, publishes scenarios |
| Late cancellation | 30 s | Cancels teams that don't show up |
| Next-missions watcher | 10 s | Feeds the lobby TVs |
| MQTT relay | event-driven | Turns Maestro notifications into Archipel state |

## The life of a session, from booking to leaving the room

Here's a team's complete journey, as seen by the stage manager:

```
 Booking         Front-desk   Archipel           Room            Maestro                 Maestro
 tool            kiosk        (site)             tablet          Game Orchestrator       Room Orchestrator
     │ bookings (5 min)          │
     ├──────────────────────────►│ session, slot
     │             │ identify ──►│
     │             │ confirm ───►│ language, players,
     │             │◄── word ────│ launch word
     │                           │ ◄─────────── RoomStateChanged(Prepared) ──────────────────┐
     │                           │ Ready → WaitingForPlayers (slot is close)                 │
     │                           │◄── start {word} ─┤                                        │
     │                           │ onboarding ─────►│ (story steps)                          │
     │                           │── ChangeRoomState(StartAsked) + team ────►│               │
     │                           │                  │                        │ Maestro session
     │                           │                  │                        ├─ start_room ─►│
     │                           │◄──────── RoomStateChanged(Started) ───────┤               │
     │                           │◄──────── SessionProgressionInit/Update ───┤◄──────────────┤
     │                           │── SSE (1 s) ────►│ interpolated countdown                 │
     │                           │◄──────── RoomStateChanged(Ended) ─────────┤◄──────────────┤
     │                           │ next session's scenario ─────────────────►│
```

Step by step:

1. **Import.** Every 5 minutes, Archipel fetches bookings and attaches them to a slot and a room. A "scenario" product (for example *Dark Magic School*, the Halloween version of our magic school) resolves to the physical room *and* its variant.
2. **Check-in at the kiosk.** The team identifies itself (email, order number, name, phone…), confirms its session, picks a language and the number of players. Archipel assigns it a **launch word**: the first word in the room's list, in the chosen language, that no neighbouring session holds, starting from an index derived from the session ID. If the team comes back to the kiosk, it gets the same word. If it declares more players than booked, it's sent to the front desk and the session is blocked for 5 minutes, so it can't simply start over with the right number.
3. **The room becomes playable.** Once the GM has finished preparing the room, Maestro confirms the *Prepared* state, which Archipel translates into *Ready*. Every 30 s, a sweep moves ready rooms whose slot is close to "waiting for players".
4. **Logging in at the tablet.** The room's tablet shows a grid of 8 words: the launch words of the sessions that can start now, padded with decoys, and never the word of a team that has already launched. The grid is built deterministically from the session ID so it stays stable while the tablet receives a state every second (client-side dedup depends on it). The team gets two attempts; after that the tablet locks for 10 s. The comment explains the choice: "Short cooldown, not a hard lock: a team that picked the wrong tablet must not leave it stuck."
5. **Story, then launch.** If the room has intro steps, the tablet plays them first, and the start order only goes out at the last one, "so the mission clock does not run while players read the instructions."
6. **Start on the Maestro side.** The game orchestrator receives the order, creates the Maestro session with the team name, player count, language and scenario, then passes the start instruction to the room orchestrator, flagged as coming from Archipel.
7. **State coming back.** Maestro publishes the state change and then the progression (initial, then roughly every 30 s). Archipel relays it to the tablet, which shows a countdown.
8. **End of game.** At the end, Archipel records it, clears the progression, and *immediately* sends Maestro the next session's scenario. The GM prepares the room, and the cycle starts again.

Meanwhile the lobby TVs show the upcoming missions; a row disappears as soon as its room goes into play.

## The contract between the two worlds

Archipel and Maestro talk over two MQTT topics:

- `/archipel/games/<room>/commands`: instructions from Archipel to Maestro;
- `/archipel/notifs`: notifications from Maestro to Archipel.

The contract is minimal. Two kinds of instruction: **change a room's state** (`ChangeRoomState`) and **play a gameflow element** (`PublishRoomGameflowCommand`: a command, a bundle, a notification or an option). Three kinds of notification: **state changed**, **initial progression**, **progression update**. Here's a start order:

```json
{
  "kind": 2,
  "room": "<Maestro room key>",
  "command": null,
  "data": [
    { "key": "language",      "value": "fr" },
    { "key": "players_count", "value": "4" },
    { "key": "team_name",     "value": "The Explorers" },
    { "key": "scenario",      "value": "halloween" }
  ],
  "session_id": 123,
  "room_state": 5
}
```

`kind: 2` means *ChangeRoomState*, and `room_state: 5` means *StartAsked*. They're integers, as everywhere in our stack, and so subject to the same rule: append, never renumber.

### Two keys for one room

A room has two identities: its Archipel code (for example `jr`), used in the topic, and its Maestro key (`jurassikroom`), carried in the message. Mixing them up was costly. The relay used to resolve the room from the Archipel code alone, and the comment tells what happened: using that key "silently dropped every state confirmation for such rooms." No error, no exception: a room whose two keys differed simply never reported its state. The relay now reads the Maestro key from the message, and only falls back to the Archipel code for single-key deployments.

### A format frozen by tests

The two sides don't evolve at the same pace: a Maestro room can run for a long time on an old package version. So the format is frozen by a test suite (`WireFormatTests`): field names and order, enums serialized as numbers, data keys, culture-invariant numbers, tolerance of unknown fields. The file header explains what's at stake better than I could:

> a changed field name or type (an enum serialized as text, a renamed data key) makes one side silently ignore the other's messages: the room no longer starts from the tablet, or the tablet no longer sees the game.

When we added scenarios, we didn't create a new message type. The scenario travels through the existing "play a gameflow element" instruction, with the `gameflow_option` type: no new contract, so no new version of the shared library to roll out on both sides at once.

## Ask, then observe

Archipel has its own state machine for each room (ready, waiting for players, onboarding, playing, ended, stopped, unavailable, GM mode…), but when the room is driven by Maestro, that state is **derived** from Maestro's confirmations:

| Maestro state | Archipel state |
|---|---|
| Preparation asked, in preparation | Preparing |
| Prepared | Ready |
| Start asked, started | Playing |
| Ended | Ended |
| Stop asked, at rest | Stopped |
| End of preparation *asked* | ignored |

The last row is the most interesting. "End of preparation asked" is only a request, and the comment says so: "only the request: Maestro can still refuse it." Archipel waits for "Prepared" before opening the room to players.

Same logic for the GM panel hidden in the tablet: when the GM asks it to prepare or stop the room, Archipel sends the order but doesn't write the state. In the code's words, "the real state is driven by the RoomStateChanged confirmations Maestro sends back", so the panel always reflects Maestro's actual state.

One last safeguard: a Maestro-driven room that has never reported a state is treated as **stopped**, not ready. Without that rule, a tablet could open to players in a room nobody had prepared.

## Launching a room: the order that must go out, and the one you undo

The start is the only moment where Archipel triggers something irreversible: the mission begins, the clock runs, the music starts. Three implementation details show what that implies.

**The order goes out even if the tablet gives up.** A tablet can reload its page, lose Wi-Fi or hit a proxy timeout in the middle of the request. So the send uses `CancellationToken.None`:

```csharp
// CancellationToken.None: the tablet may have abandoned the request (reload), the order
// must go out anyway -- otherwise the room switches to "start asked" with nothing starting.
ArchipelInstructionMessage startInstruction = ArchipelInstructionMessage.BuildStartRoomInstruction(maestroRoomKey, sessionId)
    with { Data = await BuildMaestroLaunchDataAsync(sessionId, sessionRepository, logger) };

bool published = await brokerClient.PublishInstructionAsync(startInstruction, CancellationToken.None,
    topicRoomKey: key);
```

**QoS 1, because QoS 0 was lying.** The comment explains why the send moved to *at least once*: "At most once (the builder's default) only meant "written to the socket", so a half-open connection lost the instruction while the API still answered OK." With QoS 1, the broker acknowledges receipt (PUBACK).

**If the order didn't go out, the launch is undone.** The room goes back to waiting for players, the launch timestamps are cleared, and the tablet gets a `launch_failed` error rather than a misleading "mission started". The team can try again, or call the GM.

To be honest about the limit: the PUBACK comes from the broker, not from Maestro. If the room's Maestro instance is offline, the order is accepted and the room stays in "start asked". The documented procedure is then a manual fallback from Maestro's GM console. It's a known limitation: an end-to-end acknowledgement would need a new contract and an extra wait on the tablet side.

## Time belongs to Maestro

The tablet countdown is a good illustration of the authority split. Maestro owns time: it sends an initial progression (total duration, start time) and then an update roughly every 30 s. Archipel enriches it with what Maestro doesn't know: the player count and language, taken from the session that launched the room.

Between two messages, three mechanisms keep the display honest:

- **The server ages the data.** The stored progression dates from Maestro's *last* message; Archipel brings it up to now before sending it:

  ```csharp
  // The snapshot holds the timing of Maestro's LAST message: age it to now. Sent as is, a
  // tablet re-anchored on it kept showing the same value until the next Maestro update, and
  // the countdown stopped altogether if Maestro went quiet.
  int age = (int)Math.Max(0, Math.Floor((now - snapshot.UpdatedAt).TotalSeconds));
  int remainingSeconds = Math.Max(0, snapshot.RemainingSeconds - age);
  ```

- **The tablet interpolates.** It builds an anchor (remaining time at a given wall-clock instant) and counts down locally, minute by minute and then second by second at the end, even if nothing else arrives.
- **Stale progression expires.** After 10 minutes without news it's dropped: that's the safety net against a frozen progress bar.

## Scenarios: preparing the next session

A **scenario** is a variant of a physical room: same room, same tablet, same Maestro instance, but a different product in the booking tool and a different option in the gameflow. Our magic school becomes *Dark Magic School* for Halloween; Prison Break becomes *Prison Break Asylum*.

Why not declare a second room in Archipel? Because a Maestro instance listens on a single key, and there's only one tablet state per physical room. So the scenario is a session attribute, and Archipel orchestrates how it gets applied with one simple rule: **outside a game, the scenario selected in Maestro is the next session's.**

It is sent:

- **immediately** when a game ends or the room is stopped, even if it hasn't changed (the GM may have switched it during the game);
- **by the 30 s sweep**, only if the value has changed (a session booked in the meantime, for example);
- **just before** a preparation requested from the tablet;
- **inside the start order**, as a safety net;
- **never during a game**, nor during a preparation ("the atmosphere would change halfway through"): the GM stays in control.

On the Maestro side, this option is never relayed to the room in play: it's applied to the gameflow configuration, which the next session picks up when it starts.

## Fan-out: from MQTT to the screens

A Maestro notification reaches the relevant screen within a few seconds at most (under two for a room tablet), by two different paths.

**To the room tablet.** The MQTT relay updates the state in the database (and the progression in memory). One SSE loop per tablet reads a snapshot of the room every second, shared among all readers (a single database query per room), and only sends it if it changed. The definition of "changed" has a history:

> Comparing the raw payload never deduplicated anything (serverTime alone changes every second): each tablet got the full payload every second, and since it re-anchors its countdown on each payload, the countdown froze between two Maestro updates.

The signature now excludes the fields that move every second. On the tablet, a guard rejects any message whose `serverTime` is older than the last one applied, so a late message can never take the screen back in time.

**To the lobby TVs.** The dashboards (next missions, availability) receive **events** rather than snapshots:

- each event carries an `epoch-sequence` ID, where the *epoch* is drawn at random every time the server starts;
- the last 512 events are kept in a ring buffer;
- on reconnect, the browser sends `Last-Event-ID` and the server replays whatever is missing;
- if the gap is too large, or the epoch has changed (the server restarted), the server sends a `resync` event and the client reloads everything.

On the client, a sync manager shared by every view handles the polling fallback and reconnection with backoff, and serializes every full reload so two loads never rebuild a view at the same time. A `resync` triggers a reload deferred by 2 s plus 0–2 s of extra jitter, against the thundering-herd effect when ten screens get the same signal. The TVs, which run for days on end, also do a full reload every 5 minutes, and every view reloads itself after a UI update.

Finally, on the server, availability events are published by an EF Core interceptor *after* the transaction commits, whatever the write path: no endpoint can forget to notify the screens.

## Integrating the outside world without getting stuck on it

A site orchestrator depends on services it doesn't control: the online booking tool, the MQTT broker, the Maestro instances. The rule is that none of them may block the rest.

**One invalid booking never blocks the ones after it.** In early November, one booking without a slot ID made the whole sync pass fail. Since the booking tool returns the same list on every pass, every booking listed after it stayed frozen: no new bookings, no cancellations, and teams that couldn't be found at the kiosk. The fix isolates each booking:

```csharp
foreach (var incoming in reservations)
{
    try { await UpsertReservationAsync(incoming, ct); }
    catch (Exception ex) when (ex is not OperationCanceledException || !ct.IsCancellationRequested)
    {
        _logger.LogError(ex, "… reservation {ReservationId} could not be synced, skipped until the next sync",
            incoming.ReservationId);
        _db.ChangeTracker.Clear();   // or every following SaveChanges would retry them
    }
}
```

**The broker may be missing.** The API starts even without a broker, because check-in, dashboards and admin don't need it. A single loop handles reconnection, with a 2 to 60 s backoff and a resubscribe on every reconnect. "Single" matters: an earlier version also reconnected from the disconnect handler, which "raced with it […] and turned a broker outage into a log line every 2 seconds."

**Site data is versioned.** A site's texts, rooms, GM commands and scenarios live in a dedicated Git repository as numbered patches. Archipel remembers the position of the last applied patch in a dedicated Docker volume. Before that volume existed, the position lived inside the container, and every upgrade put every patch back to "pending". A patch that fails is still marked as applied, and you fix it with a new patch: history isn't rewritten.

## Deployment

Archipel fits on one mini PC per site, running Debian 12 with Docker: PostgreSQL, Mosquitto, a migrator that must finish before the API starts, the API, the UI and an nginx gateway. CI publishes images on every merge, tagged `vYYYY.MM.DD-<sha>`. Production pins a tag; rolling back means putting the previous one back (restoring the database if a migration ran). A backup goes out every night.

The tablets and lobby screens are mini PCs running Debian 12 with Ubuntu Frame, Chromium and the on-screen keyboard as snaps, which prevents quitting the application. Snap refreshes are confined to between 3 and 5 a.m.

## What I take away from it

1. **Each level has its own authority, and it's written down.** Archipel owns the session; Maestro owns the room and the time. As long as that table is clear, conflicts sort themselves out.
2. **Ask, observe, never assume.** A higher-level orchestrator derives state from the confirmations it receives; it doesn't write it in place of whoever executes.
3. **Whatever is orchestrated must be able to do without the orchestrator.** A Maestro room plays without Archipel; Archipel checks teams in without a broker. Every dependency is optional for the side that depends on it.
4. **An irreversible order goes out no matter what, or is cleanly undone.** No "mission started" on screen when nothing went out.
5. **The contract is frozen by tests**, because the two sides never upgrade at the same time.
6. **One invalid record never blocks the ones after it.** It's obvious, and yet it caused our last sync incident.
7. **Screens must never lie.** Age the data, interpolate, deduplicate, reject what arrives out of order: the team in front of the tablet only sees the countdown, and the countdown has to be right.

Maestro brings a room to life. Archipel brings a site to life: it gets the right team to the right door, at the right time, in the right language, with the right variant of the story. Then it steps aside and lets Maestro play.
