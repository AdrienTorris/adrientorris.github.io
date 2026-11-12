---
title: "Escape rooms driven by MQTT: how I build software at Team Break"
date: 2026-11-12
lang: en
tags: [mqtt, architecture, iot, raspberry-pi, esp32, linux, debian, dotnet, systemd, distributed-systems]
---

> **Alternative titles** (pick one, then delete this block)
>
> 1. Six players, one countdown, 150 machines: the software behind our escape rooms
> 2. MQTT, Raspberry Pis and electromagnets: behind the scenes at Team Break
> 3. Building software for rooms with a body: agents, messages and Linux at Team Break

# Escape rooms driven by MQTT: how I build software at Team Break

At Team Break, a bug usually means six people locked in a room, a countdown ticking, and a trapdoor that won't open.

I build the software that runs Team Break's escape rooms. It isn't a web app behind a load balancer. It's a distributed system whose nodes are mini PCs, Raspberry Pis, ESP32s, Arduinos, Android tablets, TVs, projectors, DMX moving heads, electromagnets and smoke machines. It runs unattended across thirteen sites. It has to survive power cuts, damaged network cables and players who hit the props a little too hard. And it must never, ever put anyone at risk.

This article walks through the technical choices that follow from that: a decentralized architecture, MQTT as the backbone, a small army of specialized agents, in-house electronics, a heterogeneous Linux base, and how all of it is operated remotely.

## A room is a distributed system

Let's start with the basic unit: a room. On paper it's a 60-minute game. In practice it's a small datacenter that fits in a technical closet.

```
                        ┌───────────────────────────────────────────┐
                        │   Room central server (mini PC)           │
                        │   Debian 13 + Docker                      │
                        │                                           │
                        │  Mosquitto · Room/Game Orchestrator       │
                        │  GM API · GM UI · Agents API · PostgreSQL │
                        │  Electronic · Emergency · Healthcheck     │
                        └──────────────────┬────────────────────────┘
                                           │  MQTT (LAN, offline)
      ┌──────────────────┬─────────────────┼──────────────────┬───────────────────┐
      │                  │                 │                  │                   │
 Raspberry Pi       Raspberry Pi       ESP32 / Kincony    Arduino + W5500    Tablets, screens
 video (mpv)        audio, DMX         sensors, relays    hatches, LEDs      web kiosks
 labwc / KMS        game agents        RFID, keypads      sequencers         (Chromium, Android)
```

A central server, usually an Intel N95 or N100 mini PC, runs the Maestro stack (our orchestration engine) in Docker. Around it sit edge machines (*edge points*), each with one narrow job: play a video on a TV, play an ambient soundscape, send DMX frames to a light, read an RFID tag inside a canopic jar, count the turns of a crank.

A concrete example: *La Chocolaterie*, in Lille. It's one experience split into five software rooms, with sixteen Raspberry Pis and one central server. Its *gameflow* (more on that below) has 68 files, 588 commands, 247 notification handlers and 312 electronic I/O points. An ARP scan of the Lille room network finds 69 devices.

Across the fleet, that adds up to about 150 machines enrolled on our VPN, spread over 13 sites (company-owned and franchised), with a target of around 300 once the older rooms are migrated. Alongside Maestro, the fleet also hosts two other in-house products (Donut Quiz and Heroes Academy) and our site operations platform, Archipel.

Three constraints shape everything else:

1. **A room must work without the Internet.** The site network can go down and the VPS can be under maintenance; the game in progress must not notice.
2. **Everything restarts, all the time.** Power cuts (at Aéroville, the morning ones come from the shopping centre's evacuation drills), TVs going to sleep, sound cards dropping off the bus.
3. **People's safety does not depend on software.** Door maglocks stay hard-wired to the fire-alarm system. Software follows it; it doesn't make the call.

## MQTT, the nervous system

All these machines talk over MQTT, through a Mosquitto broker running on each room's central server.

Why MQTT rather than REST, gRPC or an enterprise bus? Because we need a protocol that everything can speak, from an Arduino Uno with 2 KB of RAM to a .NET service, by way of a Python script on a Pi and a web page in a kiosk. Pub/sub naturally decouples producers from consumers: the keypad sensor doesn't need to know who's waiting for its signal. And MQTT gives us two primitives for free that matter a lot here: *retained* messages (the last known state is handed to every new subscriber) and *Last Will* (the broker announces a client's death on its behalf).

### A topic convention

Every agent has its own inbox, built deterministically from the game, the room and the agent type:

```csharp
private static string GetBaseTopicName(GameAgentEnum agent, string game, string? room) =>
    !string.IsNullOrWhiteSpace(room) && agent != GameAgentEnum.GameOrchestrator
    ? $"/games/{game.ToLower()}/rooms/{room.ToLower()}/"
    : $"/games/{game.ToLower()}/";
// suffixes: roomorchestrator, audioplayer, videoplayer, electronics,
//           lights, dmx, customagents, gamemastering, gameorchestrator
```

That gives topics like `/games/chocolaterie/rooms/elevator/roomorchestrator` or `/games/chocolaterie/rooms/candy_store/videoplayer`. The game orchestrator, which chains rooms together, lives one level up: `/games/chocolaterie/gameorchestrator`. Next to these sit a few cross-cutting topics: client presence (`/monitoring/clients/status`), a retained health snapshot per room, and the emergency latch.

### The message is the contract

Messages are snake_case JSON. A notification for the room orchestrator looks like this:

```json
{
  "key": "stop_room",
  "sender_id": "chocolaterie-game-orchestrator",
  "sender_type_id": 8,
  "room": "elevator",
  "datas": null,
  "type_id": 3,
  "sent_at": null
}
```

`sender_type_id` is an integer: 1 for the room orchestrator, 2 for the audio player, 3 for the electronic agent, 4 for video, 11 for DMX… These integers travel on the wire and are read by agents that weren't necessarily upgraded at the same time. So the rule is strict: **new values are appended, nothing is ever renumbered**. The comment sits in the code, right above the enum.

The same discipline shows up everywhere. On the Archipel side, tests (`WireFormatTests`) freeze the JSON format exchanged with the rooms, because *a Maestro room can run for a long time on an old package version*. On our newest project the protocol is Protobuf, with the same rule written down in an ADR: fields are added, never removed or renumbered.

### QoS: game traffic vs. safety traffic

Game events go out at QoS 0, not retained. That's deliberate fire-and-forget: a sensor that drops will send its state again, and a "late" game message is worthless anyway.

Critical paths are handled differently. The emergency agent, which relays the fire-alarm system, and the health agent publish at **QoS 1, retained**. When a fire alarm goes off, the `/games/<game>/rooms/<room>/emergency` topic switches to latched and stays there. An audio player that restarts during the alarm resubscribes, immediately receives that retained message, and comes back muted. There's no race between startup and the alarm: the state lives in the broker, not in a process's memory.

Another deliberate choice: **no request/response over MQTT**. When the game master (GM) asks for a room to be prepared, the GM UI sets the state to "preparation asked" in the database, and the orchestrator does the work and then confirms over HTTP to the Agents API (backed by PostgreSQL). A room's state machine therefore rests on *asked → confirmed* handshakes, and the source of truth is a database, not a message that might have been lost.

With one deliberate exception: for briefing, end of preparation, start and stop triggered from the console, the API confirms the state in the database optimistically, without waiting for the orchestrator. A room stuck in "start asked" with six players inside is worse than a state that is briefly ahead of reality. Preparation and the end of the game are still confirmed by the orchestrator.

### What MQTT taught me the hard way

- **Keepalive.** Our in-room web pages used a 5,000-second keepalive. I built a test bench (aedes broker, simulated Wi-Fi cut) that showed a silent cut went unnoticed for **about 2 h 05 min**. At 60 s it's detected in 90 s. The reconnect backoff, which used to be a fixed 100 ms, is now progressive.
- **The agent that went deaf.** A .NET agent that reconnected didn't re-announce itself on the presence topic. It was connected, but monitoring could no longer see it. Now every (re)connection republishes presence. The first connection never gives up (backoff `1 s, 2 s, 5 s, 10 s, 30 s, 60 s`), and the retry timer uses a **monotonic clock**, because a Pi has no hardware clock and its time jumps when NTP syncs.
- **The hostname, a hidden primary key.** When machines were enrolled on the VPN, our tool renamed the hostname. Nothing happened at the time. But at the next reboot, rooms went out of service, and stayed that way for two days: rooms stuck in "preparation asked", videos present but no sound. There were two causes, both deferred until the reboot. First, our MQTT agents call `socket.gethostname()` once at startup and ignore messages addressed to any other name (Maestro addresses some machines by hostname). Second, Chromium's `SingletonLock` contains the hostname, so the kiosk refused to start, silently. Since then, the node name on the VPN is decoupled from the system hostname.

## Agents, lots of agents

Here, an **agent** is a process with a single responsibility that speaks only MQTT (plus HTTP for durable state) and knows nothing about the storyline. The video player knows how to play a segment between two cue points. It doesn't know it's showing a dinosaur.

### The .NET core

Maestro's generic agents are written in C# on .NET 8, with MediatR, EF Core and Npgsql on PostgreSQL 16, Serilog and MQTTnet 5:

| Agent | Role |
|---|---|
| Room Orchestrator | Loads and runs a room's gameflow: notifications, milestones, countdown |
| Game Orchestrator | Chains the rooms of one experience, translates GM or Archipel orders |
| GM API / GM UI | Game master console, built with native Web Components, no framework |
| Agents API | Data API (PostgreSQL) for every non-GM agent |
| Electronic Agent | Bridge between the electronics brokers (I/O panels, ESP32, Kincony) and Maestro |
| Scheduled Actions | Delayed or conditional actions, and the automatic hint system |
| Emergency Agent | Fire-alarm relay: opens doors, turns lights on, cuts sound, holds a latch |
| Healthcheck Agent | Inspects containers and devices, publishes a retained snapshot every 10 s |

One rule has no exceptions: **no game-specific code in the generic agents repository**. Every game that needs custom logic gets its own repository. For the Chocolaterie that's about a dozen agents: a home-made guitar rhythm game, a pump machine, a candy dispenser, a UV mushroom, and their web screens.

Shared code (MQTT messaging, models, gameflow loading) is published as private NuGet packages, each versioned independently: `Messaging.Mqtt` is at 1.0.146 and `Models` at 1.0.79. A package version that wasn't bumped fails a unit test. All of it is covered by about 1,800 xUnit tests in the agents and another 500 in the infrastructure libraries, including a no-mock integration test that crosses the whole chain: embedded MQTT broker → real electronics bridge → GM API SSE endpoint → relay in the UI.

### The "deported" Python agents

On the Raspberry Pis, the agents that touch hardware are written in Python and packaged as `.deb`:

- **MQTTmpv**: a libmpv video player with cue-to-cue segments, overlays, picture-in-picture, CEC control of the TVs, and multi-screen over KMS.
- **MQTTaudioplayer**: a multitrack player (pygame-ce) with fades, ducking, groups, streaming of long ambient tracks, and GM push-to-talk voice over WebSocket.
- **MQTTdmx**: DMX512 frames over an FTDI USB adapter (Enttec Open DMX clones), with loops, fades and blackout.
- Smaller agents: a sequence-puzzle engine (up to a 22-symbol Morse code in a rolling window), an offline keyword spotter (MFCC + DTW against banks of `.wav` files), and a state keeper for third-party I/O panels.

They share a common module (a git submodule) that provides a multi-broker MQTT client with reconnection and subscription replay, configuration, logging and `sd_notify` integration for systemd.

Commands are JSON, with optional filters (`PlayerID`, `hostname`, `ip`). Without a filter, the command is broadcast. Here's a video segment that notifies the orchestrator when it starts and when it ends:

```json
{
  "command": "segment",
  "data": {
    "start_cue": "MyCuePoint1", "end_cue": 7, "at_end": "pause",
    "at_start_topic": "/my_topic/start", "at_start_message": "video started",
    "at_end_topic": "/my_topic", "at_end_message": "video finished",
    "play": true
  }
}
```

## Orchestration: the gameflow

If the agents are the muscles, the **gameflow** is the central nervous system. It's a declarative JSON description of everything that happens in a room: which notifications exist, which handlers react to them, which behaviors those trigger (publish a command, validate a milestone, change the remaining time, fail the room…), which actions are scheduled, how hints are given out, and how the electronic I/O maps onto business keys.

Here's a real excerpt from the Chocolaterie, lightly trimmed: an ESP32 keypad unlocks a trapdoor and validates a milestone. The GM can get the same effect from the console.

```json
{
  "file_scope_settings": { "room": "ice_room", "electronic_proxies_format": "tb_esp32" },
  "notification_handlers": [ {
    "notifications": [ "digicode_unlocked", "milestone_state_change_r4_digicode_unlocked_validate" ],
    "behaviors": [
      { "behavior": "command_publish",    "key": "trappe_unlock" },
      { "behavior": "milestone_validate", "key": "r4_digicode_unlocked" }
    ]
  } ],
  "electronic_agent_proxies": [
    { "provider": "esp32_digicode", "type": "input",  "command_key": "digicode_unlocked",
      "label": "Digicode", "mod": "in2",  "pin": 0, "state": "off" },
    { "provider": "esp32_digicode", "type": "output", "command_key": "trappe_unlock",
      "label": "Trappe conduit", "mod": "out1", "pin": 0, "state": "on" }
  ]
}
```

A few choices worth explaining:

- **One Git repository per game *and per site*.** You'd think the same room is identical in Lille and in Grenoble. It never is: a magnet that is active-high here is active-low there. Rather than piling up conditions, each site has its own gameflow.
- **Versioning decoupled from Git.** An append-only `_versions.json` holds the version number, date, author and a description of the changes. Agents compare the latest date with their local copy. As a result, work in progress can be pushed without it reaching the rooms. The Chocolaterie gameflow has 644 entries since July 2024.
- **A local copy, pulled at startup only.** Agents fetch the latest version at boot and then work from their copy, which is what makes offline operation possible. Since several agents read the same directory, loading is serialized with an OS-level `flock`: exclusive for the downloader, shared for readers, and released automatically on a crash or a power cut.
- **Layered scenarios.** The same physical room can run a variant (Halloween, a kids' version) through a `scenario` filter and `override`s on the affected elements. The scenario is a session option, independent of the language.
- **Room tests versioned with the gameflow.** A test protocol (`_tests/*.json`), never read by the game, is validated in CI and run from a "Tests" page in the GM console, which checks each of the Chocolaterie's 312 I/O points.

One story shows that a declarative format is still code: for a long time, `"filters_behavior": "or"` was silently evaluated as AND. The cause was a constructor that assigned the parameter instead of the property. It was fixed in version 1.0.79 of the models package, with a test to go with it.

## Electronics: C on the ESP32, and lessons learned the hard way

The props are where software meets the physical world. Our standard is the **ESP32** (WROOM, 4 MB of flash) programmed in C with **ESP-IDF** on FreeRTOS. Where relays are needed, we use **Kincony KC868-A16** boards (ESP32, 16 relays and 16 inputs on PCF8574 I2C expanders, LAN8720 Ethernet) running in-house firmware. There are also Arduino Unos and Megas with Ethernet shields, and third-party I/O panels built on Raspberry Pi and Node-RED.

### A shared base for every firmware

Every prop firmware pulls in a shared base as a git submodule; the prop only adds its pinout, game logic and messages. The base builds with ESP-IDF 5.5 and 6.0 and provides about twenty components: networking (Wi-Fi with backoff, Ethernet, mDNS, NTP), an MQTT client, an HTTP server with a configuration page and live state over SSE, LittleFS storage, logs persisted to flash, a watchdog, unified GPIO handling (internal pins or expanders, debounce, filters), PN532 RFID readers (up to 8 on one bus), and WS2812 LED strips.

A few principles are baked in:

- **Hardware before network.** The boot order is fixed and documented. Outputs are initialized *before* the network, because an output must reach its safe state (locks closed, motors off) as early as possible.
- **Schema-driven configuration.** Each component declares its section of `tb_config.json` in a schema, and the web page builds its forms from those schemas. If the file is corrupted, the firmware falls back to `last-known-good`, then to `factory-defaults`, and keeps the broken file for diagnosis.
- **Safe mode.** After two failed boots in a row, the board starts in degraded mode: configuration restored, Wi-Fi and web page up, but **no MQTT**, so it stays out of the game. A boot only counts as successful after 60 minutes of uptime, so a crash in the middle of a session counts as a failure. The decision is a pure function, unit-tested on ESP-IDF's `linux` target.
- **Watchdog, core dump and signed OTA.** A 10 s task watchdog with panic, a core dump to flash, two OTA partitions, firmware and web UI updates over HTTP, and an RSA-3072-signed manifest. Builds are reproducible (`CONFIG_APP_REPRODUCIBLE_BUILD=y`) and published by CI on a tag.
- **Ethernet whenever possible.** An escape room is full of metal and of other props that emit: Wi-Fi isn't always welcome there.

Expected RFID tag UIDs live in the configuration, not in the code. Replacing a statuette lost in the Egyptian room is a configuration change, not a new firmware.

The Kincony board protocol is deliberately simple:

```json
{"command":"set","data":[{"module":"out1","pin":2,"value":"on"},{"module":"out2","pin":3,"value":"off"}]}
{"command":"get","data":[{"module":"in1","pin":3}]}
{"command":"get_all_infos"}
```

### The board that answered ping

My favorite story. The Chocolaterie's elevator hatches are driven by an Arduino Uno with a W5500 Ethernet shield. Every so often the board froze: serial port silent, game dead… yet it still answered ping. The W5500 handles ICMP on its own, so a ping proves absolutely nothing about the ATmega being alive. The Ethernet library waited on the controller in loops with no timeout, and noise from an electromagnet was enough to hang it forever.

The fix combines a 4 s AVR watchdog, W5500 retransmissions cut from ~30 s to ~1.5 s to stay under the watchdog, non-blocking MQTT, the electromagnet energized at the very start of `setup()`, and `F()` strings to bring static RAM from 57% down to 40%. Above all, the board now publishes a retained diagnostics topic with a boot counter, the cause of the last reset and a breadcrumb (`reset_step`) showing where the code was before the last restart. That's what remote diagnosis of a microcontroller looks like: leave crumbs before you die.

### DMX and stubborn fixtures

On the lighting side, MQTTdmx generates DMX512 frames in software on an FTDI FT232R: 250 kbaud, 8N2, an 88 µs break, an 8 µs mark-after-break. Some fixtures in HOLD mode produced spurious fades under continuous transmission. So we added a `send_on_change` mode that sends a **burst of 4 frames** (~130 ms) on each change: a single frame can get lost on a noisy cable, and a HOLD fixture would then keep the wrong value indefinitely.

## Linux everywhere, but not the same Linux everywhere

All of this runs on Linux, but not on *one* Linux. When I measured the fleet to design our SSH certificate authority, the breakdown was:

| Distribution | Machines |
|---|---|
| Debian 13 (Trixie) | 79 |
| Debian 12 (Bookworm) | 25 |
| Raspbian 10 (Buster) | 18 |
| Raspbian 11 (Bullseye) | 13 |
| Ubuntu 20.04 | 8 |
| Ubuntu 24.04 / 26.04 | 4 |

Add a few legacy Windows central servers that we're migrating one by one. On the hardware side: x86 mini PCs (Intel N95/N100, HP ProDesk 400 G4 DM, a Celeron N4020 with 4 GB and eMMC "on the edge of memory saturation"), Raspberry Pi 4s on arm64, even a Pi 3 B, and Debian tablets built on mini PCs. Packages are built for **amd64 and arm64**.

### The target standard

- **Central servers:** Debian 13 in server mode, no GUI, with `docker.io` and the Maestro stack.
- **Raspberry Pis:** Raspberry Pi OS Lite 64-bit / Debian Trixie arm64. For video, a golden image built with `rpi-image-gen` (pinned commit for reproducible builds) that boots straight into labwc (Wayland), then the fullscreen player.
- **Tablets and front-desk screens:** Debian 12 with Ubuntu Frame (Mir), Chromium and the on-screen keyboard as snaps, which prevents quitting the application. On some Pis, cage + Firefox.

### What heterogeneity really costs

Every distribution brings its own surprises, and most of them get handled in code:

- `paho-mqtt` 1.4.0 on Buster means keeping a compatibility shim in the shared MQTT module.
- On Bookworm, pygame 2.1.2 exposes `mixer.Channel` as a function rather than a class, so the `Channel | None` annotation crashed at import. `from __future__ import annotations` fixed it.
- On Pi 4 + Trixie, mpv's Vulkan backend (V3DV) is broken, so we force `gpu-api=opengl`.
- Debian 10's OpenSSH 7.9 doesn't understand the `Include` directive, which constrained how the SSH certificate authority was rolled out.

The real answer to heterogeneity is to **reduce the number of targets**. For that, we can now reflash a Pi remotely: the tool sends a full SD image into RAM, writes it from a small system that no longer reads the card, then reboots. First success on the bench: a Pi 4 moved from Raspbian 10 armhf to Debian 13 arm64 without anyone travelling.

### The Debian package as the unit of deployment

Everything that gets installed on a room machine is a `.deb`. That's not an aesthetic preference: it's the only format that guarantees, across all these distributions, declared dependencies, configuration preserved across upgrades (*conffiles*) and a clean uninstall.

The most interesting case is `maestro-room`. A single package, `maestro-room_<version>_<arch>.deb`, embeds every Docker image for the room as an archive that is loaded at install time. There's **no registry pull on site** and no GitHub access needed. Configuration lives in `/etc/maestro/room.env` (a conffile), and rolling back means reinstalling the previous package. CI builds both architectures with buildx, checks the package contents, tags and publishes the release.

The rules we apply to our packages were all learned from an incident:

- `apt install ./package.deb` must be enough. Every dependency must be expressible in `Depends:`. If it isn't, we change the solution: `yq` was replaced by `python3-yaml` because two incompatible implementations share that name.
- **dpkg maintainer scripts do no networking and start nothing.** Installing a package on a machine in operation must have no visible effect.
- **Versions are pinned everywhere.** No `latest` image, and the Tailscale client is frozen with `apt-mark hold`. An upgrade is a decision, never a side effect.

### systemd, my best friend and my worst enemy

Almost every agent runs as `Type=notify`, sending `READY=1` once it is actually ready. A few settings show what "soft real-time" looks like:

```ini
# Audio player
Type=notify
TimeoutStartSec=600
Nice=-10
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=50
LimitMEMLOCK=infinity
```

`TimeoutStartSec=600` has a story behind it. A catalogue of ~4,000 tracks (5 GiB) preloaded before `READY=1` exceeded the 90 s timeout: systemd killed the service, which restarted, which preloaded again… Now the player sends `EXTEND_TIMEOUT_USEC` while loading, and long tracks are streamed in one-second chunks. A 600 MiB ambient track now costs 2 MiB of RSS instead of being decoded into RAM.

The video player runs as a **user unit** (`systemctl --user`). That means `loginctl enable-linger` in the postinst (otherwise the service never starts on a machine with no login screen) and an `ExecStartPre` that waits for the Wayland or X11 socket. You can't rely on `graphical-session.target`: labwc on the Pi never activates it. Before this fix, the service started 18 s before the compositor and showed "Display not available" for 2 h 30 min.

Then there was the "the player no longer outlives its labwc" bug. With `PAMName=login`, the session script and the player live in the logind session scope, outside the service's cgroup. A `systemctl restart` killed only labwc: the old player kept running next to the new one, each holding the Jurassik room's big video in RAM, until the OOM killer stepped in on an 8 GB Pi with no swap. The fix is a few lines of shell:

```sh
/usr/bin/python3 /opt/MQTTmpv/MQTTmpv.py &
py=$!
trap 'stop_player; exit 0' TERM INT HUP
# The player must not outlive its labwc.
while kill -0 "$py" 2>/dev/null; do
    if [ -n "$SESSION_LABWC" ] && ! kill -0 "$SESSION_LABWC" 2>/dev/null; then
        stop_player; exit 0
    fi
    sleep 2
done
```

One last number: a service with `Restart=always` had restarted **4,655 times** without anyone noticing. Reporting `NRestarts` in our monitoring is the next step. An "active" service isn't necessarily a healthy one.

### Video rendering without a desktop

On general-purpose machines we prefer **direct KMS** rendering (`vo=gpu`, `gpu-context=drm`) to a graphical session: fewer packages, and no tty taken over. For multi-screen setups, a single process holds the *DRM master* of `/dev/dri/card0` and drives several connectors, each with its own `PlayerID` and VA-API decoding. It can't be any other way: a DRM card accepts only one master. An output that drops is recreated with exponential backoff, and the service only exits when every output is dead.

Even memory is a negotiation. The player forced a 2 GiB demuxer cache in both directions: with a 1.6 GB video, RAM grew with every game until it froze an 8 GB server. We went back to mpv's defaults, with one deliberate exception: the Chocolaterie elevator, where **five screens** are synchronized with each other and with the sound *with no synchronization system at all*. Only keeping the whole video in RAM makes that work.

### Docker yes, Kubernetes no

Docker is used only on the central server, with Compose and `live-restore` enabled. The question of container orchestration on room machines was settled in writing: *a room runs business software, not a cluster.*

Power-cut resilience is handled at the systemd level:

- `maestro-room.service` runs `docker compose up -d`, with a bounded restart limit so that a real failure stays visible.
- A recovery service detects when dockerd restarted "dirty" (ports still allocated) and does a down/up cycle, at most once per boot.
- A watchdog timer restarts unhealthy containers every minute, at most 3 times per service per hour, **never during a game** and never during an upgrade (maintenance lock).
- All of it is validated on a "power cut" bench: a throwaway Debian 13 VM that gets killed abruptly with SysRq.

A small operations tip: since Pis have no hardware clock, we compare boots with `journalctl -o short-monotonic` rather than wall-clock time.

## Running 150 machines across 13 sites

I can't travel every time an audio player misbehaves. Remote operations are therefore a product in their own right, called *vigie*, also shipped as two Debian packages (server and edge).

- **A self-hosted VPN that is strictly additive.** We self-host Headscale (pinned version), and machines join the tailnet with the Tailscale client. The VPN touches neither the DNS, nor the routing, nor the interfaces of a machine. Every `tailscale up` option is passed explicitly, even at its default value, to neutralize any leftover preferences:

  ```bash
  tailscale up --login-server "$login_server" --auth-key "$authkey" \
    --hostname "$node_name" --accept-dns=false --accept-routes=false \
    --advertise-routes= --advertise-exit-node=false --advertise-tags= \
    --ssh=false --timeout 60s
  ```

- **Naming by function, not by machine.** `<product>-<site>[-<level>…]-<function>[<n>]`, for example `maestro-lille-chocolaterie-ascenseur-tv4`. A name describes a function at a location: a replacement Pi takes the name of the one it replaces.
- **Push-based monitoring.** Each machine reports its state every minute through a systemd timer (with a randomized delay to smooth the load). With pull, a machine that's off makes everyone wait. With push, *an offline machine signals itself by its silence*. A node's identity is its tailnet address: no per-machine token, and removing a node from the VPN is enough to silence it.
- **Short-lived SSH certificates** (16 h) issued by an internal certificate authority, instead of keys copied by hand onto every machine.
- **No automatic corrective action.** Monitoring informs; a human decides.

For diagnosis, I built a maintenance knowledge base from two months of interventions: 58 symptoms, 27 procedures, one sheet per site. A few rules apply to every ticket: find the *real* time of the game, read the logs before the code, and **never act on a room while a game is in progress**. Of the 58 symptoms, 20 can be fixed with a closed action (restart, re-arm), 10 need someone on site, and 16 are code issues. The value is in the diagnosis, not in the restarts.

Safety has its own procedure: never trigger the emergency to test it, and after a real alarm, explicitly re-arm the audio players. The day nobody did, the Chocolaterie elevator stayed silent for seven hours. The retained latch was doing exactly what it was designed to do.

## Archipel: wiring bookings into the rooms

Archipel is our site operations platform: sync with the online booking tool, a front-desk check-in kiosk, an in-room tablet for players, and dashboards on the lobby TVs. It runs on one mini PC per site, on .NET 10 with PostgreSQL 16. The front end is vanilla JavaScript (ES modules, no npm, no bundler), fed by SSE with a polling fallback.

The bridge with Maestro naturally goes through MQTT. Archipel sends instructions (`ChangeRoomState`, publishing a gameflow) and receives notifications (`RoomStateChanged`, `SessionProgressionInit`, `SessionProgressionUpdate`). The countdown on the in-room tablet sums up the philosophy: Maestro is the authority on time. It sends an `Init` (total duration, start time) and then an `Update` about every 30 s, and the tablet **interpolates** the countdown itself from a wall-clock anchor. No per-second message ever crosses the network.

A recent lesson: one booking that arrived without a slot ID blocked the whole sync pass, which runs every 5 minutes. As a result, no new bookings, cancellations or payments were imported, and teams couldn't be found at the kiosk. The fix is a classic we forget too often: **one invalid record must never block the ones after it**. Each booking is now processed and retried independently.

Our newest project, a multi-game room, pushes these principles further: a FastAPI server, games written as pure functions `(state, action) → (new state, effects)`, a game protocol in Protobuf over WebSocket with sequence numbers and a full snapshot on reconnect, MQTT for control (presence via Last Will, retained state, QoS 1 commands), and one constraint at the top of the requirements: it must work **without the Internet**.

## What about AI agents?

In an article about agents, it would be a shame not to mention the other kind. A good part of the code described here was written in pair with Claude Code, and the way we work has adapted:

- **Every repository has a `CLAUDE.md`** describing the vocabulary, the design rules and the known pitfalls. On our newest repositories, one rule is explicit: *documentation is the only shared memory of the agents working on the repository*. Tests check that every path quoted in the docs exists and that no page exceeds 150 lines.
- **A three-role "software factory"**: a planner agent writes a spec with testable acceptance criteria, a human approves it, a builder agent implements it, and a verifier agent tries to break it, with at most three fix loops. Tests carry the IDs of the acceptance criteria ("CA-24").
- **Guardrails written after incidents** for AI in operations: an explicit human go-ahead before any write on site, one room per tool call, no blocking loops during live debugging (a smoke machine still remembers), and *no initiative*, after DMX channels were changed "to test". When the assistant goes through our maintenance MCP server, per-site permissions are enforced server-side: we don't rely on the system prompt alone.

## What I take away from it

If I had to sum up what this playground has taught me:

1. **The message is the contract.** Wire formats only grow by addition, and integers are never renumbered. A room can run for years on an old version.
2. **Everything restarts, all the time.** Every component must republish its real state on reconnect, operations must be idempotent, and critical state must live in the broker (retained) or in a database, not in a process's RAM.
3. **Hardware first, network second.** An actuator must be in a safe state before the board even knows it has an IP address.
4. **Silence is information.** Push monitoring, short keepalives, Last Will: you want to know when something is dead, not just when it's alive.
5. **A Debian package, not a how-to.** If installation needs a manual step, that step will be forgotten on the twentieth machine.
6. **Don't touch a room mid-game.** Not monitoring, not the watchdog, not me.
7. **People's safety stays hard-wired.** Software follows the fire-alarm system; it doesn't replace it.

Building software for escape rooms is distributed computing where every node has a body: a motor, a lock, a speaker. It's demanding, often frustrating (a board that answers ping while it's dead…), and I don't know many playgrounds that teach you as much. And when a team walks out of a room cheering, I know that behind the door, dozens of agents and hundreds of MQTT messages did exactly what was expected of them.
