# Five years at Team Break: from Donut Quiz to Maestro

*A look back at the projects I led between 2021 and 2026.*

My time at Team Break is coming to an end. Before moving on, I wanted to take stock of what we built: the products, the platforms, the technical choices, and the mistakes we learned from.

## In short

- **Path:** fixed-price contractor (2021 – 2023), then employee (March 2023 – November 2026).
- **Role:** architect and lead developer of the software that runs our rooms. I also wrote the specifications, deployed on site, wrote the documentation and ran technical hiring.
- **Scope:**
  - a dozen escape games run by Maestro, across nine sites;
  - Donut Quiz, on nine sites;
  - Heroes Academy, on three sites including one in Quebec;
  - Archipel, the connected reception system at Aéroville.
- **Core stack:** .NET (5 through 10), ASP.NET Core, EF Core, PostgreSQL, MQTT, Docker, Raspberry Pi.

## The context

At Team Break, we run escape games and immersive experiences: four company-owned sites (Lille, Paris 9, Aéroville, La Défense) and a franchise network stretching from Rouen to Grenoble and all the way to Brossard, Quebec.

Behind every room there is software:
- scenarios to orchestrate;
- dozens of electronic puzzles to drive;
- screens, sound and lights;
- game masters who need proper tools;
- bookings that have to line up with all of it.

I spent these years designing, building and running that software.

| Period | Status | Projects |
|---|---|---|
| 2021 – 2022 | External contractor (Pepit), fixed-price | Donut Quiz v1, Heroes Academy v1 |
| 2023 | Employee from 1 March | Heroes Academy central kiosk, Headquarters, GameEngine |
| 2024 | | Satisfaction survey, birth of Maestro, the Chocolaterie and Walking Dead on Maestro |
| 2025 | | Maestro at Aéroville and franchise sites, Heroes Academy v2, first Archipel prototype |
| 2026 | | Donut Quiz v2, Archipel in production, Maestro industrialisation, migration of Lille and La Défense |

---

## 1. Donut Quiz: where it all began (2021 → 2026)

Donut Quiz is a quiz show played in a room. Six teams compete from touch-screen desks, each named after a donut flavour and fitted with a real buzzer and an LED bar. A big shared screen, the "Saloon", brings the game to life. DMX spotlights, a moving head, a smoke machine and soundtracks set the mood. Mini-games, special strikes and surprise events punctuate the game, all the way to the award of the Golden Donut.

**As a contractor (2021 – 2022)**, I built v1 end to end on a fixed-price basis:
- the game server and the "Game Factory", which assembles each game's question set (fully rewritten in 2.1);
- the desk and Saloon apps, in .NET 5 / Avalonia on Raspberry Pi, with LibVLCSharp for video;
- each site's administration and check-in (ASP.NET Core, MongoDB);
- the central question back-office and its nightly sync with the sites;
- the hardware integration: DMX over FTDI, GPIO buzzers, HifiBerry sound cards.

One room means eight Raspberry Pi 4s (six desks, the Saloon, the game server), plus one server per site. To roll this out at scale, I set up Azure DevOps pipelines with self-hosted agents on every site. I also wrote a 70-page operations manual. More than once I found myself on site, debugging a silent buzzer or a stubborn sound card.

**As an employee (2023 – 2026)**, I kept the product evolving:
- version 2.2: pause, short mode, "joke answer"…;
- a "Central Atmosphere" component that drives the moving head and smoke;
- the Game Master split into two components, the Sound Master and the Spotlights Master;
- new installations: Paris 9, Lille, Grenoble.

In 2026 I led **v2**, an MQTT architecture built on our audio and DMX agents shared with Maestro. I deployed it myself in Lille, Aéroville, La Défense, Caen, Paris 9 and Rouen between February and June. A hardening sprint followed:
- about twenty audio bugs fixed;
- desk LEDs moved to MQTT;
- content filtering.

We then started the move to .NET 10 and web-based screens. Donut Quiz now holds more than 3,300 audio files per room and runs on nine sites.

## 2. Heroes Academy: a physical course driven by software (2022 → 2026)

Heroes Academy is a one-hour team course through a series of physical challenge rooms: ropes, "Snake" (a floor of light-up tiles), "Don't Touch The Floor", basketball hoops, communication panels, and a final quiz Arena. Screens show instructions, timer and score. Results feed a per-site leaderboard and a public one.

**v1 (contractor, 2022).** I built it on a fixed-price basis, and it has been in production in Lille since July 2022. Stack: .NET 5, a Razor Pages admin, Avalonia screens on Raspberry Pi, MongoDB, electronics driven over MQTT and GPIO expanders. Within ten months, more than 2,200 teams and nearly 8,000 players had gone through it.

**Employee (2023 – 2025).** In March 2023 I built the **central kiosk** in four days. It aggregates scores from every site and feeds the public leaderboard on the website. I then evolved v1 on my own:
- the Tarbes installation;
- a gameplay overhaul in early 2024 (Snake algorithm, scoring weighted by team size);
- rebuilding a server after a hardware failure;
- proper versioning in 2025.

**v2 (Sept. 2025 → 2026).** This was a complete rewrite, most of which I wrote (over 80% of the commits):
- .NET 8, PostgreSQL, MQTTnet 5;
- a central Ubuntu server, and one Raspberry Pi per room running the room orchestrator, the electronics bridge, the screens and the audio player;
- configuration served by the API and cached on every machine;
- a whole site installed with a single command;
- more than 820 unit tests.

v2 let us open new sites, starting with **Brossard, Quebec**, in January 2026. Lille switched to v2 in autumn 2026, with a new **Arena mode** selected at build time. More additions followed:
- gameplay settings stored in the database;
- cameras and push-to-talk for game masters;
- **scenarios** (Halloween, Christmas, hardcore…).

## 3. GameEngine: the prototype that paved the way (2023 → 2024)

My first big project as an employee was giving Team Break its own engine to run its rooms. That was **GameEngine**, codename "CHARLIE". Each room carried the entire stack on a Raspberry Pi:
- an engine executing a JSON *gameflow*;
- a session dispatcher;
- the game master's API and UI;
- a multitrack audio player;
- sync with a central back-office.

GameEngine ran real rooms:
- **Le Trésor des Pirates** at La Défense, in October 2023;
- **Los Bandidos** in Terville, then at La Défense and Aéroville;
- **Égypte** at La Défense.

Above all, it taught me what not to do:
- gameflows relied on numeric identifiers, which made them unreadable and fragile;
- every format change hit both the JSON and the database schema while rooms were in production;
- complex puzzles ended up as hard-to-maintain specific code.

In spring 2024, rather than push on with a v2, I chose to start over from a clean slate.

## 4. Maestro: the platform (May 2024 → today)

Maestro was born in May 2024 with a simple ambition: run any room reliably and uniformly, and give game masters polished tools. We needed a platform that was more maintainable and more stable, with gameflows a human can actually read. It is the project I am proudest of.

### Architecture

Maestro is a set of .NET 8 agents designed with DDD, communicating over MQTT (Mosquitto):
- the **room orchestrator** runs the gameflow as a state machine (preparation, briefing, in game, ended, at rest…);
- the **game orchestrator** chains sessions across several rooms;
- the **game master API and UI** are built with ASP.NET Core minimal APIs and Web Components, no framework;
- the **electronic agent** is a configurable proxy that translates the MQTT formats of the various puzzle vendors (A+ Props, Bela/Node-RED, Kincony, our own ESP32 boards…);
- **audio**, **video** and **DMX** agents are reused from one game to the next;
- **scheduled actions**, **emergency** (fire-safety relay), **healthcheck** and **reporting** agents feed our central server and our booking system.

The core holds only generic code. Each game has its own repository of specific agents, and shared code ships as private NuGet packages.

### The gameflow at its heart

Each game, and each site, has its own JSON gameflow versioned in Git, organised by puzzle rather than by technical type.
- Agents download the latest version at startup and keep a local copy so they can run offline.
- A versions file decides what goes live.
- A patch system means nobody ever edits the database by hand.

This readability is what let several of us write and ship rooms.

### Industrialisation

We started from manual installs: systemd services, archives copied over scp. By 2026 we had:
- a `maestro-room` Debian package embedding the Docker images, built in CI for amd64 and arm64;
- a migration tool for legacy installs;
- versioned releases, from v1.1 in July to v1.18 in November 2026;
- a watchdog and automatic recovery, added after a power cut once took a room down for three hours.

### Adoption

Today Maestro runs **a dozen games across nine sites**, nearly forty game × site combinations: Walking Dead, the Chocolaterie, Tombeau du Pharaon, Trésor des Pirates, Los Bandidos, Haunted Room, Magic School, Jurassik, Koh-Lanta, Prison Break, Fort Boyard and Lost.

On top of these comes the **Lobby**, a site's reception area run as a room in its own right: looping soundscape, mission briefings, light scenes on the room facades.

The rollout came in waves:
- **summer 2024:** the Chocolaterie and Walking Dead in Lille;
- **late 2024:** Grenoble and La Défense;
- **2025:** Aéroville;
- **2025 – 2026:** the franchise sites (Bourges, Terville, Metz, Rouen);
- **autumn 2026:** the rooms in Lille, Paris 9 and La Défense that still ran on the vendor's controller. I led the Lille migration.

The last GameEngine rooms, at La Défense, were replaced in October 2026: we had come full circle.

## 5. La Chocolaterie: our most ambitious game (2023 → 2026)

La Chocolaterie is five connected rooms, sixteen Raspberry Pis and a wealth of puzzles:
- a candy store, with a cash register and candy jars;
- a chocolate garden, with a pump machine, a Twister, a whack-a-mole, and even a "marriage proposal mode";
- an elevator, with Guitar Hero, bass shakers and hatches;
- an ice room, where players play a two-level platform game controlled by real buttons, fans and pipes;
- an inventing room, with a candy dispenser, a recipe book, a lemon press and an LED floor.

I first designed it in 2023 on GameEngine. In May 2024 it became the first game rebuilt on Maestro. It served as the platform's test bed and pushed it to its limits:
- more than 600 commands, nearly 300 notification handlers and over 300 electronic proxies in the gameflow;
- 341 revisions between v0.1.1 in July 2024 and November 2026;
- a platform game I rewrote in Godot / C# in 2025.

I was its architect and main developer.

## 6. Archipel: the connected venue (2025 → 2026)

Maestro runs a room. **Archipel** runs the whole site: it is the venue layer above the rooms, in production at Aéroville since spring 2026.

The player journey:
1. The booking comes in from 4Escape.
2. At reception, a check-in kiosk finds the booking by name, email, phone or order number. It confirms the language and player count, then gives the group its world, its locker instructions and a **secret launch word**.
3. Screens announce upcoming missions and availability, with last-minute deals.
4. At the door, a tablet wakes up and the team picks its word.
5. Archipel sends Maestro the start order (team, language, player count, scenario), then relays progress and the countdown back.

**Technical choices:**
- .NET 10, ASP.NET Core minimal APIs, EF Core, PostgreSQL, MQTTnet 5;
- a front end in native ES modules, no framework, with real-time screens over SSE;
- pull-only deployment of immutable Docker images, on one mini PC per venue;
- a versioned MQTT contract with Maestro, published as NuGet and locked down by wire-format tests;
- venue data (worlds, missions, rooms, words) versioned in Git as patches validated in CI.

I wrote the core backend and nearly all the specifications. We plugged Donut Quiz in as a second game system.

A reliability campaign wrapped up the project: a test bench, 61 acceptance scenarios and CI on every repository. It ended with the Aéroville update on 4 November 2026.

## 7. Headquarters: the central server

Headquarters, our "HQ", is Team Break's central server. I reworked it three times:
- **2023:** a .NET 7 back-office. It pulls in data from the Donut Quiz and Heroes Academy sites and produces per-room and per-venue reports. Alongside it, GameEngine had its own Headquarters, which pushed gameflows to the rooms.
- **2024:** a narrower product, put into production: the **satisfaction survey**. After their game, customers rate their experience. A discount voucher is generated in 4Escape and emailed through Mailjet, and the happiest customers are invited to leave a Google review.
- **2026:** a fresh start in .NET 10 that collects the sessions reported by every Maestro instance, with one API key per venue.

## 8. Beyond the code

**Documenting to hand over.** I wrote:
- operations manuals and per-room sheets;
- central-server runbooks;
- scoping documents with numbered decisions;
- a technology radar and a Git branching guide, to give us a common frame.

**Hiring.** In 2024 I designed our technical interview: a code review of a CodeNames implementation, a database modelling exercise, and questions on LINQ and async.

**Making the team autonomous.** For a long time I was the only developer. My goal was always for our tools to let us write and ship rooms without depending on me: readable gameflows, written procedures, reproducible deployments. In 2026 I was no longer Maestro's top contributor, and that is probably what I am proudest of.

**Working with AI.** In 2026, AI became part of our daily work as developers.
- I adopted a workflow where I write the issue, which serves as the specification. An agent (GitHub Copilot, then Claude Code) produces the pull request, which I review, fix and merge.
- In autumn 2026 alone, that meant more than 200 pull requests on the game repositories.
- We then turned this practice into a pipeline: planning, human approval of the spec, implementation, then verification by a second agent.

Well framed, these tools don't replace the engineer. They give back time for architecture, specifications, tests and documentation.

## What I take away

- **Readability is a feature.** The move from GameEngine to Maestro was first and foremost about gameflows a human can read, review and hand over to someone else.
- **The field is always right.** A power cut, a worn-out SD card, a flaky venue network: resilience (local copies, endless reconnection, watchdogs) is not a luxury.
- **Industrialise early.** Packages, images, versioned releases and written procedures are what take you from one room to forty.
- **Data belongs in Git.** Gameflows, venue configuration and reference data are versioned, reviewed and validated in CI.
- **Know when to throw away.** Dropping GameEngine for Maestro was the best technical decision of these years.

## Stack and skills

- **Architecture:** event-driven distributed systems (MQTT), DDD, CQRS / mediator, state machines, versioned message contracts.
- **.NET (5 through 10):** ASP.NET Core (minimal APIs, Razor Pages), EF Core, MediatR, MQTTnet, Avalonia, xUnit / MSTest, private NuGet packages.
- **Data:** PostgreSQL, MongoDB.
- **Front end:** Web Components, native ES modules, Server-Sent Events.
- **Hardware and embedded:** Raspberry Pi, GPIO, DMX, integration of ESP32 and puzzle-vendor boards, Godot (C#).
- **DevOps:** Docker, GitHub Actions, GHCR, Debian packages, systemd, Azure DevOps.
- **Practices:** specifications and scoping documents, acceptance testing, operations documentation, technical hiring, AI-assisted development.

## Thank you

Thank you to Team Break for its trust, from that first fixed-price contract to today. Thank you to the game masters and site teams, who lived through the early versions and patiently reported every bug. And thank you to my colleagues on the tech team for everything we built together.
