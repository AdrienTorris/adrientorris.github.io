---
title: "Donut Quiz: behind the scenes of a connected quiz room"
description: "Nine Raspberry Pis, thirteen screens, sound, DMX lighting and physical buzzers. What it took to build the Donut Quiz room, and the rewrite that finally made it stable: goodbye Avalonia, Akka.NET replaced by MQTT, .NET 10, Debian packages and reproducible Raspberry Pi images."
lang: en
tags: [dotnet, mqtt, raspberry-pi, sse, dmx, iot, architecture, lessons-learned]
---

# Donut Quiz: behind the scenes of a connected quiz room

Donut Quiz is a Team Break game room. Several teams compete in a fast-paced quiz, each behind its own desk, with a big shared screen, sound and lights. The rounds have pastry-themed French names (*Mise en bouche*, *Pour du beurre*, *Apéro*, *Surprise du chef*, *Milkshake de frites*, *Cerise sur le gâteau*), teams throw special moves at each other (*Bouchée double*, *Pique-assiette*, *Retard en cuisine*…), and the game ends with an awards ceremony.

For players, it's an hour of fun. For me, it's a distributed system of about ten machines that has to keep its timing to the second, with no IT staff on site, in venues spread across France.

This post covers the hard parts of building that room, and then the rewrite that made it **more extensible, more stable and much cheaper to maintain**: dropping Avalonia, replacing Akka.NET with MQTT, moving from .NET 5 to .NET 10, and deploying through Debian packages and reproducible Raspberry Pi images.

> **TL;DR**
> - The desks are no longer Avalonia apps. They are framework-free HTML/CSS/JS pages, shown by a browser in kiosk mode and fed by Server-Sent Events.
> - Akka.NET is gone, replaced by an MQTT broker. The game server, the screens, the audio player (written in Python), the DMX agents and the buzzers all speak the same protocol.
> - The whole solution moved from .NET 5 to .NET 10 (LTS), with central NuGet package management. It went from 44 to 30 projects and from 31 to nearly 2,000 tests.
> - The sync between the head office and the venues was a project in its own right, which I cover in [a dedicated post](./donut-quiz-sync-en.md).
> - Zip-based Azure DevOps releases were replaced by versioned, reversible `.deb` packages, per-role Raspberry Pi images, and a remote 32-bit to 64-bit migration of the Pis that required no site visits.

---

## The room in numbers

| Item | Details |
|---|---|
| **Desks** | 6, named after flavours: *chocolat*, *citron*, *caramel*, *fraise*, *pistache* and *myrtille*. Each one has a touch game screen, a score screen, a physical buzzer and an RGB LED bar. |
| **Saloon** | A big public screen that shows the atmosphere and the rankings. |
| **Lighting** | DMX512 spotlights, plus the desk LEDs. |
| **Sound** | Room speakers with three track groups (ambient, voices, effects). About 3,400 tracks per room. |
| **Machines** | 9 Raspberry Pi 4 per room (6 desks, saloon, "donut", server), for a total of **13 screens**. |
| **Media** | 13 to 19 GB of images, GIFs, videos and sounds, synced from the head office. |
| **Fleet** | 9 venues, company-owned and franchised. |

## Why this is harder than it looks

On paper, it's a quiz. In practice, several constraints pile up.

- **Real time shows, and you can hear it.** A voice line one second late, two sounds playing over each other, an LED stuck on red, a frozen desk: the whole room notices. In a team game, perceived unfairness is worse than the bug itself.
- **Nobody technical on site.** Game masters run the room, technicians install it, and it is maintained remotely. A diagnosis that requires SSH and reading JSON isn't a diagnosis.
- **Off-the-shelf hardware.** Raspberry Pis, SD cards, GPIO, USB-DMX dongles, touch screens whose EDID advertises nonsense, different spotlight models in each venue, hand-made wiring.
- **A long lifespan.** The software has to survive new Raspberry Pi OS releases, framework end-of-life dates and hardware replacements.
- **Offline by design.** The room must play on its local network. The head office only supplies content (questions, themes, media), which is synced to each venue.

---

## Version 1: what worked, and what cost us

The first version of the game was built by a contractor: my former company, where I was a managing partner. It ran on this stack:

- **.NET 5**, with **Avalonia 0.10** UIs on the desks and the saloon (LibVLCSharp for video, a dedicated control for animated GIFs);
- **Raspberry Pis running 32-bit (armhf) Raspbian Buster** with X11;
- **Akka.NET** for communication between components, plus a **Hello Service** for discovery: every component announced itself to it, so the game server knew which IP address to reach each one on;
- **MongoDB** for data;
- **Azure DevOps** for source control, CI and deployment, chosen because the team knew it well and it was free for up to five users. A build pipeline produced a self-contained bundle, then per-venue and per-room *release pipelines* ran on a **self-hosted agent installed in each venue**. The infrastructure inventory (each component's IP address, volume coefficient, DMX offset…) lived in versioned Excel files that the pipelines read.

The components had self-explanatory names: *Game Server* (the engine), *Game Master* (sounds and spotlights), *Desk*, *Shared Screen* (the saloon), *Administration*, *Synchronization*, *Check-in* and *Hello Service*.

It worked. But every site slowly turned into a special case, and I have a few field memories from that time.

- **.NET 5 was not an LTS release.** One day, the deployment bundle could no longer be built: the NuGet certificates of SDKs older than 5.0.202 had expired, and since this wasn't an LTS release, they were never renewed. The fix was to move the build agent from Ubuntu 18.04 to 22.04 and bump the SDK. Lesson learned: never ship a non-LTS runtime in a product that has to live for years.
- **A buzzer stopped being detected** after a Raspbian update, because of a bug in the .NET IoT library. I had to switch from logical pin numbering (GPIO 21) to physical numbering (pin 40), then deploy the hotfix desk by desk. The pin number was hard-coded, so a single faulty pin would have required a custom build. It is configurable now.
- **No sound in the room.** The deployment process was installing a HifiBerry sound card overlay on a Pi that used the jack output.
- **The browser blocked the admin port.** The admin site listened on port 10080, which Chrome and Firefox consider "unsafe". The documentation had to grow an appendix explaining how to allow it, browser by browser.
- **The release of Raspbian Bullseye** forced us to pin Buster, which the tooling no longer offered by default.

None of these issues was hard to fix. What was expensive was **the cost of every fix**: a build, a release, a pass over every machine, and a configuration that drifted a little further from one site to the next.

---

## Akka.NET → MQTT: pick a protocol, not a framework

On paper, Akka.NET was very appealing: an actor model, location transparency, and a game server sending orders to desks looks a lot like an actor messaging other actors.

In practice, three limits showed up.

1. **Remoting is point-to-point.** Every node has to know the host and port of the others. Hence the Hello Service, the IP inventory in Excel, and a room reconfiguration every time a machine was replaced.
2. **Everyone has to speak .NET and Akka**, with compatible versions and serialization formats. The hardware ecosystem wanted something else: an audio player in Python, small DMX agents, and eventually microcontrollers (ESP32) for the buzzers.
3. **It's hard to observe.** There was no way to "listen in" on what the room was saying.

MQTT solves all three.

- **There is a single address to know**: the Mosquitto broker running on the room server. The Hello Service is gone.
- **Publish/subscribe** fits the need exactly. The server publishes one message per desk, one for all desks, and one for the saloon.
- **It's polyglot**: C#, Python, ESP32 firmware, everyone speaks MQTT.
- **It's observable.** `mosquitto_sub -v -t '/games/dq/#'` shows the whole game live.

| Topic | Direction | Purpose |
|---|---|---|
| `/games/dq/orchestrator` | screens → game server | commands and player input (answer, registration, special move…) |
| `/games/dq/desks/{colour}` | server → one desk | what that desk should display |
| `/games/dq/desks/all` | server → all desks | broadcast |
| `/games/dq/saloon` | server → saloon | public screen |
| `/games/dq/desks/{colour}/led` | server → DMX agent | the desk's LED bar colour |
| `/games/dq/lights` | server → DMX agent | room spotlights |
| `/games/dq/audio` | server → audio player | play, stop, fades |
| `/games/dq/desks/{colour}/buzz` | buzzer → server | buzzer pressed |

The resulting architecture principle is simple: **the game server is the single source of truth.** It publishes DTOs that describe what to display, play or light up, and consumers just execute them. Hardware drivers were moved out of the business code and handed to small dedicated agents:

- **MQTTaudioplayer** (Python) plays tracks through ALSA;
- **MQTTdmx** turns messages into DMX frames on a USB-FTDI dongle, with one agent per bus;
- **the buzzer** reads a GPIO pin and publishes to `/desks/{colour}/buzz`. Tomorrow it will be an ESP32 on **the same topic**, and neither the server nor the screens will notice.

### What MQTT won't do for you

- **QoS.** By default, MQTTnet publishes at QoS 0 ("at most once"). On a busy room network, a lost `play` means silence and a lost `stop` means two sounds at once. Every audio command is now published `AtLeastOnce`, and the commands are idempotent so duplicates are harmless.
- **Reconnection.** Every client has to reconnect forever and resubscribe, even if the very first connection failed. Another trap: in MQTTnet 4.0, `ConnectAsync` waits for the CONNACK with no bound at all. A frozen broker then leaves a client "connected" with no real session. So I wrote an `MqttConnectionKeeper` with a liveness probe.
- **Wildcards.** Technical topics (screen supervision, see below) live under a separate root, `/screens/dq/`. If a third-party agent ever subscribes to `/games/dq/#`, it will never receive a message it doesn't expect. The risk is ruled out by design, not by assumption.

---

## Goodbye Avalonia: the desks are just browsers now

Moving to .NET 10 meant migrating from Avalonia 0.10 to Avalonia 11, a major breaking change, on top of LibVLC on ARM and X11. Re-reading the desk code, I made the discovery that unblocked everything: **the desks were already thin clients.** The server published a JSON `DeskMessage`, and the Avalonia ViewModels were nothing more than a state machine that turned a message into a screen.

So the UI had a single job: render a JSON DTO. The web does that very well.

### The choices

- **HTML, CSS and JavaScript, no framework.** I ruled out Blazor: it needs either the WebAssembly runtime or a permanent SignalR circuit, which is too heavy for a Raspberry Pi and for such a simple need.
- **One central web host.** A single ASP.NET Core application, `GameScreens.Web`, serves every screen in the room. Each Pi runs Firefox in kiosk mode under Wayland (labwc).
- **Two surfaces per desk**: `/desk/{colour}` for the touch game screen and `/desk/{colour}/score` for the score. Both subscribe to the same stream.
- **An MQTT → SSE bridge.** An `IHostedService` subscribes to the desk and saloon topics, keeps the latest state of each screen, and pushes it to the browsers as Server-Sent Events. Player input goes back through a plain `fetch` POST, republished to `/orchestrator`. **The game server didn't change a single line.**

Why SSE rather than WebSockets? Almost all the traffic flows from the server to the screen. SSE is plain HTTP, reconnects natively, and `curl` is enough to debug it. The rare player inputs fit a POST perfectly.

```csharp
// GameScreens.Web — SSE stream for one screen (simplified)
public static async Task StreamAsync(HttpContext ctx, ScreenStateStore store, string screenKey, CancellationToken ct)
{
    ctx.Response.Headers.ContentType = "text/event-stream";
    ctx.Response.Headers.CacheControl = "no-cache";
    ctx.Response.Headers["X-Accel-Buffering"] = "no";

    var channel = Channel.CreateUnbounded<string>(new() { SingleReader = true });
    void Handler(string key, string json) { if (key == screenKey) channel.Writer.TryWrite(json); }

    store.Subscribe(Handler);
    try
    {
        await ctx.Response.WriteAsync(": connected\n\n", ct);
        await ctx.Response.Body.FlushAsync(ct);

        // Replay the latest state along with its age, so a video resumes at the right moment.
        // State older than 5 minutes is never replayed: the screen stays on its splash.
        if (store.TryGetLatest(screenKey, out var latest, out var elapsedMs) && elapsedMs <= StaleReplayCapMs)
            await WriteEventAsync(ctx, InjectResyncElapsed(latest, elapsedMs), ct);

        while (!ct.IsCancellationRequested)
        {
            var wait = channel.Reader.WaitToReadAsync(ct).AsTask();
            if (await Task.WhenAny(wait, Task.Delay(KeepAliveInterval, ct)) == wait)
            {
                if (!await wait) break;
                while (channel.Reader.TryRead(out var json)) await WriteEventAsync(ctx, json, ct);
            }
            else
            {
                // Every 15 s: a comment (for curl probes) THEN a named event.
                await ctx.Response.WriteAsync(": ping\n\nevent: ping\ndata: 1\n\n", ct);
                await ctx.Response.Body.FlushAsync(ct);
            }
        }
    }
    catch (OperationCanceledException) { /* client gone */ }
    finally { store.Unsubscribe(Handler); }
}
```

### Reproduce before you improve

The most important rule of the port was this: **the web screen must behave exactly like the Avalonia screen.** For every interaction, I read the legacy code first, then wrote the web code. The "improved" behaviours I could have invented, like submitting on tap or mapping the buzzer to the space bar, are bugs for players and game masters who know the game. In the original game, **a tap selects an answer, which can still be changed, and the buzzer submits it** together with the response time. The Donut Quiz logo is itself a clickable buzzer.

### Porting the visual identity

- The shapes (trapezoid speech bubbles, cards, header banner) were Avalonia `Polygon`s derived from SVG. They became SVG files again, injected inline so that `fill: var(--…)` follows the theme.
- The eight themes are sets of CSS custom properties. The Montserrat font was kept.
- **Text that was too long** was a customer's first remark after the migration: answers and explanations spilling out of their bubbles. The font-fitting helper existed, but it wasn't wired to the answer tiles. It also had to be hardened twice: retry on the next `requestAnimationFrame` when the box has no width yet, and measure again after `document.fonts.ready`, because a measurement taken with the fallback font concluded too early that "it fits".
- **Video sync.** When a screen (re)connects, the replayed state carries `resync_elapsed_ms`. A video slide then resumes at the right moment, in step with the saloon and the other desks, instead of restarting from zero.

### Media

Each desk used to carry its own copy of the media, more than 13 GB per machine. Today the media lives in a single Docker volume on the room server and is served over the LAN with `Cache-Control: public, max-age=31536000, immutable`. Since content is addressed by id, it never changes. The browser's disk cache, on the SD card, does the rest.

### Hardening a kiosk

Disabling pinch-zoom and two-finger swipes, falling back to the splash screen instead of a black screen, mapping touch input to the right output when a Pi drives two displays, defaulting to 1280×720 when a screen advertises bogus modes… None of this is glamorous, and every item came from a field report.

---

## "Connected" doesn't mean "alive"

This is probably the most important lesson of the rewrite.

The trap is the half-open TCP connection: a switch, an access point or a NAT table drops the connection without telling anyone. The browser then keeps waiting for bytes that will never come, `readyState` stays `OPEN`, no `onerror` fires, so `EventSource`'s built-in reconnection never kicks in. A screen can go deaf without a single error showing up anywhere. Worse, the server's keepalive was an SSE **comment** (`: ping`), and comments don't trigger any JavaScript handler. The page had no way to tell "nothing to say" from "dead link".

The fix comes in three layers.

1. **On the server**, a *named* `ping` event goes out every 15 seconds, in addition to the comment.
2. **In the browser**, a liveness guard reopens the stream after 45 seconds without a single byte, which is three missed keepalives. Reconnection uses exponential backoff capped at 30 seconds, so that thirteen screens don't turn a briefly slow server into a reconnection storm. And two `EventSource` instances never coexist.

```javascript
// stream.js (excerpt)
export const LIVENESS_TIMEOUT_MS = 45000;    // 3 missed server keepalives
export const RECONNECT_MAX_DELAY_MS = 30000; // slow down, never give up

function armGuard() {
  livenessTimer = cancel(livenessTimer);
  livenessTimer = setTimeout(scheduleReconnect, LIVENESS_TIMEOUT_MS);
}

function scheduleReconnect() {
  if (closed || reconnectTimer !== null) return;
  livenessTimer = cancel(livenessTimer);
  if (es) { es.close(); es = null; }          // never two open streams at once
  const delay = Math.min(RECONNECT_MAX_DELAY_MS, 1000 * 2 ** failures++);
  reconnectTimer = setTimeout(() => { reconnectTimer = null; connect(); }, delay);
}

function connect() {
  es = new EventSource(url);
  es.onmessage = (ev) => { onActivity(); /* … */ };
  es.addEventListener('ping', onActivity);    // the keepalive becomes observable
  es.onerror = () => { if (es.readyState === EventSource.CLOSED) scheduleReconnect(); };
  armGuard();
}
```

3. **In the other direction**, each screen answers every ping with a *pong* published over MQTT. The pong carries the surface (game or score), because a desk whose score screen alone has died must not show up green. The admin site therefore knows the real state of all 13 screens, and the server automatically restarts a desk that has dropped off.

The first version of this monitoring counted open sockets on the server. It relied on four layers of indirection (in-memory counter, JSON over HTTP, HTTP probe, bash parsing), and **two of them lied while their tests were green**. A ping/pong round trip proves in a single measurement what the counter couldn't: the JavaScript is running, it receives, it answers, and its answer arrives.

---

## Sound: where every mistake is audible

The audio model is easy to describe. Three track groups (`ambient`, `voices`, `effects`). A `play solo` replaces the tracks of the same group. A `notify` plays a cue while lowering the ambient track (*ducking*), then restores it. Fades tie it all together. The player is a Python program (pygame) driven over MQTT on the "donut" Pi.

The symptoms were just as easy to describe: **overlaps and silences, "every now and then".** The investigation uncovered eight separate causes, split between the game server and the player. Here's my favourite:

```csharp
// Before
foreach (var audioInstruction in audioInstructions)
{
    if (audioInstruction.Offset > 0)
    {
        _ = Task.Run(async () =>
        {
            await Task.Delay(audioInstruction.Offset, cancellationToken); // token never cancelled upstream
            await RunAudioInstructionsAsync(audioInstruction, cancellationToken);
        }, cancellationToken);
        return; // ← silently drops ALL the remaining instructions of the step
    }
    await RunAudioInstructionsAsync(audioInstruction, cancellationToken);
}
```

That's a `return` that should have been a `continue`, plus a fire-and-forget task that no game stop ever cancelled. A delayed instruction could therefore fire over the next step, or even after a `stopall`. In the fixed version, each delayed instruction gets a token that is cancelled when a game stops or loads, its exceptions are logged, and the loop carries on.

The other causes:

- QoS 0 publishing, covered above;
- a `FadeOutBackground` sent every time, even for a question with no voice-over: the ambient track dropped to 20% for 30 to 90 seconds, which the room heard as dead air;
- on the player side, ducking applied before the `play` and never restored if playback failed.

Then there was the content pipeline. Every sound is converted to a clean WAV by ffmpeg at upload time (in a Hangfire job), and the admin site publishes a `/config/audio` manifest so the player can fetch its own tracks.

---

## Lighting: painless DMX512

DMX is disarmingly simple: 512 channels, each with a value from 0 to 255, sent in a loop, with no notion of a destination address. Each fixture listens to N channels starting at its own start address.

The game server models **light storyboards** (game start, right or wrong answer, awards…) and publishes `one_shot`, `fade`, `loop` or `blackout` messages. The MQTTdmx agent, one per bus, keeps the 512-channel frame and sends it to the USB-FTDI dongle.

- **Desk LEDs.** Each desk has its own controller, and therefore its own bus. All six desks use the same start address, and that's not a typo: there is no shared chain to offset.
- **Spotlights** sit next to each other on a single bus: `channel = GlobalOffset + (n − 1) × BufferSize + Index`. The *Index* (a colour's position inside a fixture) comes from the manufacturer's manual. The *Offset* depends on the on-site wiring.
- **A classic field bug**: `GlobalOffset = 0`. DMX starts at channel 1, so the agent ignored channel 0 and the first spotlight stayed dark.
- **Robustness**: an `Index` that overflows `BufferSize` (a typo in a hand-edited file) is now ignored for that single function, instead of dropping the whole frame.
- **The strobe is disabled, for good.** It's a game design decision, not a setting to rediscover on site.

A single configuration file, `/etc/donutquiz/gameserver-dmx.json`, replaces the two previous settings: the offset in each Avalonia desk's appsettings and the configuration of the old "spotlights master". And when two colours light up together on a desk, being able to inspect the frames remotely clears the software and sends the technician straight to the terminal block.

---

## Head office ↔ venue sync

A room has to keep playing even if the venue's internet connection drops. So each venue has its own database and its own media: content flows down from the head office, and played games flow back up. Between a watermark that followed the wrong clock and properties silently vanishing behind a 200 OK, this work deserves its own post: **[Donut Quiz: syncing a head office and its venues without silently losing anything](./donut-quiz-sync-en.md)**.

---

## From .NET 5 to .NET 10, and a big clean-up

- **Central package management.** The solution had 44 projects with diverging package versions (`System.Device.Gpio` 1.3.0 here, 2.2.0 there). A single `Directory.Packages.props` now centralizes the versions. The policy is simple: freeze the DI container (Autofac) with `VersionOverride` so behaviour doesn't change, and align trivial packages upwards.
- **A direct jump from `net5.0` to `net10.0`**, with no intermediate .NET 8 stop. The real breaking changes were few but sneaky: the `win10-x64` RID became `win-x64`, MSTest 2 was incompatible, `UseHsts()`, and above all a missing `AddEnvironmentVariables()` that **silently ignored** the entire Docker configuration.
- **Testing against real content.** Instead of mocking questions, the full stack runs in docker-compose and pulls real content from the head office. A mock gives a false sense of a milestone reached. Real content reveals the real migration bugs.
- **The clean-up.** The Avalonia apps, the old spotlight driver, the test console apps and dead commented-out code were deleted. The solution went from **44 to 30 projects**.
- **Tests went from 31 to nearly 2,000.** The GitHub Actions CI builds the solution, runs the unit tests and the JavaScript screen tests (`node --test`), then brings up an **end-to-end validation bench** in Docker (MongoDB, Mosquitto, screens, one head-office instance and one venue instance) and runs the validation tests against it.

---

## Deploying to Raspberry Pis… without going there

### From Azure DevOps to GitHub, from zip files to Debian packages

With Azure DevOps, a release meant dropping a zip on each machine, unzipping it, restoring the executable bit and writing a systemd unit by hand. It was all driven by pipelines reading IP addresses from an Excel file, with one self-hosted agent per venue and access tokens that expired at the worst possible time. It held up for a handful of rooms, but none of it was reproducible at the scale of a growing network.

The code now lives on GitHub, and every component is a **versioned Debian package**. A package activates nothing on install: it prepares the configuration and ships three tools, `dq-cutover` to switch over, `dq-rollback` to go back, and `dq-doctor` for a self-verifying health check. Switching over is explicit and reversible.

- **Server services run in Docker.** The .NET binaries are cross-compiled as *self-contained* `linux-arm64`, then shipped with a COPY-only Dockerfile. The `docker build` runs **natively on the Pi** at first boot: no QEMU, no buildx, nothing to compile on the target.
- **Hardware services run natively.** The kiosk, the buzzer (GPIO), audio (ALSA) and DMX (FTDI) run as systemd services. They need direct hardware access, and there's no reason to containerize them.

### Golden images per role

The Pis are built from **one image per role** (desk, saloon, donut, server), generated with `rpi-image-gen`: Debian 13 "Trixie" arm64, a minimal base with no desktop, Wayland with labwc, and pinned versions for reproducibility.

Each machine's own configuration lives in a file on the **FAT boot partition**, so it can be edited from any Windows PC without booting the Pi:

```ini
# /boot/firmware/donutquiz.conf
ROLE=desk                          # desk | saloon | donut | host
SERVER_URL=http://<server-ip>:5003
COLOR=caramel
```

On first boot, a *oneshot* service reads this file and derives everything it can: the buzzer topic from the colour, the broker address from the server URL. It generates the config files, maps touch input to the game screen, sets the hostname and enables the services. In the field, the procedure boils down to: **flash, fill in three keys, plug in.**

Finally, the kiosk package is **the same for legacy X11 and the new Wayland setup.** It detects the display backend at runtime:

| Action | X11 (Buster) | Wayland (Trixie) |
|---|---|---|
| List outputs | `xrandr` | `wlr-randr` |
| Place one window per screen | `wmctrl` | labwc `<windowRules>` |
| Map touch input | `xinput map-to-output` | labwc `<touch mapToOutput>` |

### 32-bit to 64-bit, remotely, without bricking anything

One big problem remained: the whole fleet ran **32-bit (armhf) Buster**. A `dist-upgrade` doesn't change the architecture, and an in-place major upgrade of Raspberry Pi OS isn't supported. So everything had to be reflashed… dozens of SD cards spread across France.

For that, I wrote `dq-migrer-ab`, a script that migrates a Pi **remotely, using an A/B scheme**:

1. **Probe** (read-only): role, addresses, SD card health, which process actually opens the DMX dongle, and generation of `donutquiz.conf`;
2. **Shrink**: a single boot into an initramfs, through `tryboot`, shrinks the Buster root partition and creates two partitions behind it, `p3` (Trixie boot) and `p4` (Trixie root);
3. **Write** the image to `p3`/`p4` *from the running Buster system*, with sha256 verification;
4. **Prepare**: configuration, fresh SSH host keys, media, VPN identity;
5. **Trial**: a trial boot into Trixie. Until the switch is committed, **a normal reboot, or even a power cut, brings the Pi back to Buster**;
6. **Commit**: a separate command, never chained automatically, makes Trixie the boot partition.

The only risky window is the partition shrink, which lasts a few minutes. That's why SD card health is checked beforehand: a worn-out card that never came back from the initramfs taught me to do so. Remote access goes through a mesh VPN (Headscale, the self-hosted Tailscale control server), which replaced the old remote-desktop tool.

---

## What I take away from this

- **Choose LTS releases.** Always, for a product that has to last.
- **Thin clients and a single source of truth.** That's what made it possible to replace Avalonia without touching the game server.
- **A protocol beats a framework** when heterogeneous devices need to talk. MQTT replaced Akka.NET and the Hello Service, and it opened the door to Python and the ESP32.
- **"Connected" isn't "alive".** Measure an end-to-end round trip rather than an intermediate indicator, and be wary of green tests that validate a mechanism rather than an outcome.
- **Software in containers, hardware drivers native.**
- **Put field configuration where a technician can edit it**: on a FAT partition, not in a JSON file you have to edit over SSH.
- **Keep every step reversible**: package rollback, A/B trial boot.
- **Reproduce before you improve.** A successful migration is, first and foremost, one that nobody notices.

---

## And today?

The room runs. Most venues have switched to the new stack, games follow one another, screens reconnect on their own, and an intervention that used to require a trip is now handled remotely. Above all, the players' feedback speaks for itself:

<!-- Google review screenshots -->
![Google review 1](./images/google-review-1.png)
![Google review 2](./images/google-review-2.png)
![Google review 3](./images/google-review-3.png)

What's next is already on the table: ESP32 buzzers, a single image for every role resolved through `donutquiz.conf`, and more tools so the head office and franchisees can manage their content without me. But the room is no longer a permanent construction site, and that was the whole point of this rewrite.
