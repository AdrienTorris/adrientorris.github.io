---
title: "Heroes Academy (1/3): designing software for a physical game venue"
description: "A venue where people climb, jump and run is a real-time distributed system. A look back at V1: .NET 5, Avalonia, MongoDB and Raspberry Pis, and what it taught me."
lang: en
tags: [dotnet, csharp, mqtt, raspberry-pi, iot, architecture, lessons-learned]
---

# Heroes Academy (1/3): designing software for a physical game venue

> **The Heroes Academy series**
> 1. **Designing software for a physical game venue** (you are here)
> 2. [The V2 rewrite for Canada](./heroes-academy-2-v2-rewrite.md)
> 3. [Eight months of AI-assisted development](./heroes-academy-3-ai-assisted-development.md)

In some projects, a bug gives you a stack trace. In others, it gives you a team of six players stuck in a room in front of a frozen screen, while the next team waits at the door.

Heroes Academy is the second kind. It is an "active game" concept: teams go through a series of physical rooms where they climb ropes, jump on a light-up floor, shoot baskets or cross a room without touching the ground. Everything is timed, scored, narrated by voice-overs and accompanied by videos. There are three Heroes Academy venues today: two in France, in Lille and Tarbes, and one in Canada, in Brossard, near Montréal.

I have been developing the software for these venues from the start. This series covers the technical challenges in three parts: the first version and its lessons (this article), the full rewrite done for the Canadian venue, and the role AI-assisted development came to play.

---

## A game venue is a distributed system

From the outside, a Heroes Academy is a sequence of rooms. From the code's point of view, it is a real-time distributed system. It runs on a dozen machines per site, in an environment that looks nothing like a datacenter.

A team follows a **route**: Dressing, Briefing, Warm Up, then the game rooms (Ropes, Snake, Don't Touch The Floor, Logs, Balls, Communication), then the Debriefing. Each game room lasts a few minutes and contains some or all of the following:

- backlit **target buttons** to hit, sometimes high up;
- **magnetic-lock doors**, which lock the entrance during the game and release the exit at the end;
- **beacons** and ambient lighting;
- a **screen** for instructions, timer, score and videos, and **speakers** for music, voice-overs and sirens;
- depending on the room, a **floor grid of tiles** (Snake), **floor sensors** (Don't Touch The Floor) or **basket sensors** with point displays (Balls);
- a **secondary mission**: hidden green buttons that unlock a hint video, followed by a multiple-choice question.

The resulting constraints look nothing like those of a web application:

| Constraint | Technical consequence |
|---|---|
| Rooms are **chained** | One stuck room blocks the whole route, and every team behind it. |
| The game is **physical** | A lost button press, or one counted twice, is noticed immediately. |
| The hardware is **embedded** | Raspberry Pis on SD cards, power cuts, a switch turned off by mistake. |
| On-site staff are **not technical** | A Game Master must be able to restart a room without a terminal. |
| Sites are **far away** | Lille, Tarbes, and soon Montréal: nobody can just drop by and reboot a Pi. |

### Electronics: MQTT rather than GPIO

The very first structural decision concerned the electronics. The earliest versions drove I/O straight from the Raspberry Pi: GPIO and MCP23017 I²C expanders polled every 50 ms. That code still exists in V1… fully commented out.

We quickly moved to **networked I/O boards** that speak MQTT. Each board publishes its inputs and receives its outputs as very simple text messages:

```text
/io/input   ←  /c0m2p5/on     (controller 0, module 2, pin 5: button pressed)
/io/input   ←  /c0m2p5/off    (button released)
/io/output  →  /c1m0p3/on     (light the matching target's LED)
```

This is probably the decision that aged best. The software no longer needs to know *how* a button is wired: it receives strings. That means a whole room can be simulated with `mosquitto_pub`. This decision survived the rewrite.

---

## V1: one monolith per Raspberry Pi

V1 was started by a contractor: the services company where I was then a managing partner. It was developed on Azure DevOps before the code moved to GitHub in 2023. It opened Lille, then Tarbes. Its architecture fits in one diagram:

```text
                 ┌────────────────────────────────────────┐
                 │  Central server (Windows)              │
                 │  ASP.NET Core 5 (admin, Razor Pages)   │
                 │  MongoDB                               │
                 └───────────────▲────────────────────────┘
                                 │  long-lived MongoDB connections
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────┴─────────┐   ┌─────────┴────────┐    ┌─────────┴────────┐
│ "Ropes" Pi       │   │ "Snake" Pi       │    │ "Balls" Pi       │  …
│ Avalonia 0.10    │   │ Avalonia 0.10    │    │ Avalonia 0.10    │
│ logic + UI +     │   │ logic + UI +     │    │ logic + UI +     │
│ audio (LibVLC) + │   │ audio (LibVLC) + │    │ audio (LibVLC) + │
│ MQTT, 1 process  │   │ MQTT, 1 process  │    │ MQTT, 1 process  │
│ local Mosquitto  │   │ local Mosquitto  │    │ local Mosquitto  │
└────────▲─────────┘   └────────▲─────────┘    └────────▲─────────┘
         │ MQTT                  │ MQTT                  │ MQTT
    I/O boards              I/O boards              I/O boards
```

Each Pi ran **a single Avalonia desktop application** on .NET 5, published self-contained for `linux-arm`. It contained everything: game logic, UI, audio (LibVLC with four parallel players for ambience, voices, effects and siren), and the MQTT client for the local broker the I/O boards talked to.

### MongoDB as the integration bus

There was no message bus between the rooms and the server. **The database played that role.** Every second, each Pi asked MongoDB whether it had a session to play:

```csharp
// V1, simplified: every Raspberry Pi polls MongoDB once per second
var session = gameSessionService.GetSession(roomId);   // SlotStart <= now < GameEnd

bool isRecovering = session.TimeElapsed != TimeSpan.Zero
    && session.TimeLeft > session.GameEndDateTime - DateTime.Now + TimeSpan.FromSeconds(6);

if (isRecovering)      current.Recover(session);                       // restarted mid-game
else if (isPlaying)    current.ComparePersistentDataSession(session);  // admin edited the session
else                   current.Start(session);
```

In the other direction, every game wrote its remaining time back to the database **every second**. Game Master actions ("skip mission", "reset room") were just flags set on the session document, which the Pi discovered on its next tick.

It wasn't elegant, but it worked, and it was easy to reason about. The `Recover` logic even let a room resume a running game after a crash, by recomputing the remaining time from the scheduled end time.

It did have a hidden cost: correct behaviour depended on **each Pi's system clock** and on a **permanent database connection**.

### The zombie process

This is the most frequent incident in V1's whole history, and it almost always happened in the morning, before opening.

The central server restarts, because of an overnight Windows update or a power cut. When it comes back, no room works, and the only known fix is to **reboot every Raspberry Pi**.

For a long time we suspected MQTT. We were wrong: each Pi has its own Mosquitto, and a server reboot cannot kill it. The real cause was found by watching the TCP connections opened from a Pi. The application kept a **pool of long-lived connections to MongoDB**. The server reboot killed those sockets, and the client never rebuilt the pool.

The detail that made the problem nasty: **the process didn't die**. It kept running, attached to dead sockets. Each Pi did have a relaunch script:

```bash
# ha-keepalive.sh, started with the graphical session
while :; do
    $dq_app_path & pid=$!
    wait $pid
    sleep 2s
done
```

But that script relaunches a *dead* process, not a *useless* one. The fix took the form of a watchdog. It probes the database port every 15 seconds and, on an "unreachable, then back" transition, kills the game so that `ha-keepalive` restarts it:

```bash
reachable() { timeout 3 bash -c "exec 3<>/dev/tcp/$MONGO_HOST/$MONGO_PORT" 2>/dev/null; }

while :; do
  if reachable; then
    ok=$((ok + 1))
    if [[ $state == down && $ok -ge 4 ]]; then   # 4 successful probes in a row
      restart_game                               # kill, then ha-keepalive relaunches
      state=up
    fi
  else
    ok=0; state=down
  fi
  sleep 15
done
```

The safeguards are deliberate:

- a room whose database never went away is never touched;
- a network hiccup of a few seconds triggers nothing;
- the log is truncated at 256 KB, because these machines run on SD cards.

The lesson goes beyond this case: **a live process is not a working process.**

### The other pain points

The zombie process was the most visible problem, but not the only one:

- **A frozen stack.** .NET 5 has been out of support since May 2022, and V1 was pinned to Avalonia 0.10. Upgrading meant migrating the runtime, the UI framework and the LibVLC integration on ARM all at once, with no safety net. Avalonia itself is not to blame. The problem was putting it at the centre: when the UI framework also hosts the game logic and the audio, everything is tied to its lifecycle.
- **Concurrency.** Several timers ran in parallel (session polling, countdown, per-game timers), on top of MQTT callbacks and the UI dispatcher. To avoid concurrent access, some games used `Monitor.TryEnter(padlock, TimeSpan.Zero)`. A button press arriving while the lock was held was therefore **silently dropped**. Snake, for its part, ran multi-second `Thread.Sleep` calls for its animations.
- **Hand-made deployments.** The process was the same every time: `dotnet publish`, delete the config files, zip, copy to the site, unzip, restore the local configuration. A room's wiring (which button on which pin) lived in **three different places**, and they drifted apart: an output remapped by hand to work around a dead pin, a site file copied from another site. Some differences between sites were even handled by commenting or uncommenting a line before compiling (`// For Lille` / `// For Tarbes`).
- **Media.** Videos were committed to the Git repository (over 600 MB of `.git`, including a 321 MB video that GitHub rejected). Full-screen playback spawned an external `vlc` process, which was later killed by name.
- **The Windows server.** The admin was a console application launched from the Startup folder. It therefore depended on a user logging in, and on an ASP.NET development certificate that eventually expired. One day a switch was turned off by mistake: fourteen machines went down at once, and the admin didn't come back when power returned.
- **Real hardware.** In Tarbes, the Balls room sensors sit at the top of the basket. A ball bouncing on the rim therefore scored twice, and we had to add a one-second per-basket debounce. None of this appears in any specification.

### The Arena, or the art of keeping teams apart

V1 also introduced the **Arena**, a shared room where each team spends 30 seconds between two game rooms answering a question on a touchscreen. On paper, it's simple. In practice, the Arena is shared by every team on the route, and two teams must never meet there.

The scheduler pre-computed a team's entire route when the session was created, one step per document. We had to rework the spacing between starts (Briefing + Dressing + 3 × Arena) and add unit tests over 30 to 50 consecutive sessions to guarantee that no crossing remained possible.

### A V1 that did its job

It would be unfair to remember only the problems. V1 opened two sites and kept them running for years. It also evolved a lot through contact with the field:

- Balls baskets weighted by the number of players;
- a redesigned Snake, with random sequences and a dedicated scenario for teams of six;
- voice-overs at halftime, one minute and ten seconds before the end;
- questions read aloud in the Arena.

Some settings were made, unmade and remade within a few days, directly on site. That's the nature of a physical game: you don't know whether a mechanic is fun until you've watched a team play it.

---

## What V1 taught me

1. **A database is not a message bus.** Polling a shared database creates hidden coupling, to the clock as much as to the connection. The day the connection dies without the process dying, nothing moves anymore, and nothing reports it.
2. **A live process is not a working process.** Supervising a PID is not enough; you must supervise the function.
3. **Whatever can drift will drift.** Configuration copied to three places ends up with three versions. A site difference handled with a comment ends up compiled for the wrong site.
4. **The field is always right.** Bouncing balls, a switch turned off, a server rebooting at night: no spec replaces an hour in the room.

When the Canadian project took shape, the question was simple: install yet another copy of this V1 more than 5,000 km away, with no way to drop by and reboot a Pi in the morning? The answer was no. That is the subject of part two.

---

> **Next in the series**
> 2. [The V2 rewrite for Canada](./heroes-academy-2-v2-rewrite.md): V2's architecture, principles and bugs.
> 3. [Eight months of AI-assisted development](./heroes-academy-3-ai-assisted-development.md): what the Git history says about AI in the project.

*Code excerpts in this article are simplified. Addresses, credentials and site-specific infrastructure details have been deliberately left out.*
