---
title: "Heroes Academy (3/3): eight months of AI-assisted development"
description: "What the Heroes Academy Git history says about AI in development: when it arrived, what it produced, how I frame it, and what it cannot see."
lang: en
tags: [ai, claude-code, github-copilot, dotnet, testing, methodology, lessons-learned]
---

# Heroes Academy (3/3): eight months of AI-assisted development

> **The Heroes Academy series**
> 1. [Designing software for a physical game venue](./heroes-academy-1-v1.md)
> 2. [The V2 rewrite for Canada](./heroes-academy-2-v2-rewrite.md)
> 3. **Eight months of AI-assisted development** (you are here)

The first two articles in this series cover the design of Heroes Academy: a V1 that learned the hard way what a distributed system inside a gym looks like, then a rewrite built for a site six hours away.

This third part is about something that changed the way I work during that rewrite: AI-assisted development. I won't make predictions. I'd rather stick to what the project's Git history says: when AI arrived, what it produced, how I frame it, and what it can't do.

---

## What the numbers say

I counted as "AI-assisted" any commit written by an agent, or carrying an assistant's `Co-Authored-By` trailer. Here is what that gives on V2's main repository, excluding merge commits:

| Period | What was happening | Commits | AI-assisted |
|---|---|---:|---:|
| Sept 2025 → Feb 2026 | V2 architecture, Canada opening | 643 | 0 |
| Mar → May 2026 | First experiments, with GitHub Copilot | 196 | 27 |
| Jun → Sept 2026 | Claude Code, Arena mode, Lille migration | 160 | 74 |
| Oct → Nov 2026 | Game scenarios, the Switchboard | 121 | 118 |

Three things stand out in this table.

**AI arrived after the foundations.** V2's first 643 commits, the ones that lay down the whole architecture described in [part two](./heroes-academy-2-v2-rewrite.md), were written without an assistant. They cover MQTT everywhere, no database in the rooms, games as step machines and hardware described as data. Canada opened on that code.

**The switch was fast.** From 2 assisted commits in March 2026, we went to 118 out of 121 in October-November. Overall, 100 of the 169 pull requests merged into the main repository come from agent branches (`claude/*` or `copilot/*`). The share is similar, or even higher, on the game UI, the admin UI and the documentation.

**Tests exploded.** To me, this is the most important number:

| Date | Automated tests |
|---|---:|
| January 2026 (Canada opening) | ~10 |
| March 2026 | 17 |
| June 2026 | 161 |
| September 2026 | 217 |
| November 2026 | 842 |

The test suite grew fiftyfold in eight months. I'll come back to this below.

---

## The tools, and what I used them for

### GitHub Copilot: from issues to pull requests

The first experiments, in spring 2026, used Copilot's coding agent. I opened an issue, it opened a pull request. That format suits well-scoped bugs:

- the **8 pm in Montréal bug** (the dashboard filtered on the UTC day) was fixed with a rolling 24-hour window, with tests on the boundaries;
- the **bonus answer ignored** during the reveal window, a race between two threads;
- a **session slot** computed without taking an ongoing Warm Up session into account;
- an **XSS vulnerability** in the admin scoreboard;
- and a first wave of **unit tests for all nine games**, which explains the jump from 17 to 161 tests in spring.

A few fixes also came from Copilot Autofix, applied to review comments.

### Claude Code: in sessions, and inside GitHub

From June 2026, I mostly worked with Claude Code, in two ways:

- **in interactive sessions**, for the big projects: Arena mode, game scenarios, the Switchboard, the one-command deployment tool, the browser-based test UI;
- **inside GitHub**, through a GitHub Action. An `@claude` mention in an issue or PR triggers the agent, and **every pull request gets an automatic review** before I look at it.

---

## The method: write before you code

What changed most is not how fast code gets written. It's what I write *before* the code.

### A plan, alternatives, decisions

Arena mode is a good example. Before a single line of code, two design documents were written with the agent. The first drove the mode from the database. The second made it a compile-time constant. Both carried the same header: *"design plan, no implementation at this stage"*.

The reference plan included a table of **locked decisions**, for example:

- an Arena after every game room except the last;
- the question's video stays in the game room, and only the answer moves to the Arena;
- the question's result is carried by the Arena row.

These are business decisions, and I made them. I picked the compile-time variant because a site never changes mode.

Implementation then followed the plan in phases, each in its own PR:

1. session creation without overlaps;
2. a visit identity carried end to end;
3. sequencing by position;
4. the Arena game itself;
5. reporting.

### Increments, not big bangs

The Switchboard, the Communication room's new gameplay, was split into seven increments, each in its own PR:

1. foundation and gameplay selection through configuration;
2. a playable prototype;
3. coded calls, then blurred images and general knowledge;
4. priority and deferred messages;
5. audio feedback and ambience;
6. settings and profiles;
7. general shutdown and instructions.

The game scenarios (Halloween, Christmas, "hardcore" mode) followed the same split. An increment must be small enough for me to genuinely review it, and complete enough to be tested in the room.

### Specify first, fix second

The story of the "reconnected but deaf" `SessionsOrchestrator`, told in part two, has an epilogue that sums up the way I work.

A first fix had been produced quickly by the agent at the end of July: resubscribe to topics after reconnecting. It was **set aside**. Instead, I had a specification written with about twenty acceptance criteria. Several of them relied on constraints only a human knew:

- **the code base**: the deliverable had to start from the commit matching the production version, not from `main`, so that it shipped this fix and nothing else;
- **the actual deployment**: the binary is deployed **without** its `appsettings.json`, because the server's older file is kept. The new settings therefore needed defaults in code, which one criterion checks explicitly;
- **operations**: a fallback to the standard MQTT port after three failures on the configured port.

The code was rewritten from that spec, with 32 tests driving a mocked MQTT client and a fake delay provider. No test actually waits for a second.

That's the lesson I keep: **the agent writes code fast; the value is in the acceptance criteria.**

---

## Tests became (almost) free

On V1, as at the start of V2, tests were rare: a few on scheduling, about ten when Canada opened. A physical game is validated in the room first, and there was no time left for the rest. With an agent, writing a test costs almost nothing. What costs is reviewing it.

Two architectural choices made this possible:

- **time is injected everywhere** (`TimeProvider`, an abstract delay provider). A seven-minute game, a reconnection loop with backoff or an Arena with teams following each other are tested in a few milliseconds, deterministically;
- **hardware is just MQTT.** Simulating a button press means publishing a string.

The result: over 800 tests, including regression tests written at the very moment each bug was fixed (Arena crossings, ghost sessions, ignored answers). These tests are also what lets me have an agent modify the code: they are the net under the trapeze.

---

## Documenting for humans… and for agents

An effect I hadn't anticipated: documentation became an asset again. An agent reads the same documentation as a technician, and follows it more scrupulously.

- The repository contains a **deployment skill** (a Claude Code *skill*). It describes how to install or update a site, which questions to ask before acting (game mode, addresses, access), and what never to do, such as inventing a room key.
- The documentation repository was reorganised into **one folder per site**, with the history of interventions. What each folder contains serves the person on site as much as the agent assisting them.

AI is also used outside the application code. The watchdog that fixes V1's zombie process, described in [part one](./heroes-academy-1-v1.md), was written with it, along with the document that explains the cause from the TCP connections observed on a Pi. So was bringing back a Windows admin server that went down after a network outage.

---

## What AI cannot see: the room

The agent writes code for the system you describe to it. The field tells you whether the description is right.

The Switchboard plan says it in black and white: before writing the engine, **measure a magnet's bounce** with `mosquitto_sub` by removing and putting back a single magnet on a panel. Same for the actual latency between the electronics bridge, the central broker and the orchestrator: it has to be measured before fixing any timing windows. No model knows how the magnets in *our* room bounce.

The "no beacon during play" rule came from watching players, who leave the room as soon as a beacon lights up. So did the basket sensor mounted at the top in Tarbes, which made a bouncing ball score twice. That information exists in no repository until a human writes it down.

And the agent makes mistakes, of course. During the Arena mode PR series, it introduced a build error in the tests (optional arguments inside Moq expressions), fixed right after. Arena mode also lived on a separate branch for a while, and every change had to be ported back to `main`. The agent does that mechanical work well, but each port is another PR to review.

---

## What changed, and what's still missing

In eight months, a very small team delivered, among other things:

- Arena mode and the Lille migration;
- game scenarios;
- the Switchboard;
- a one-command deployment tool;
- a browser-based test UI;
- a test suite fifty times larger.

Without an assistant, that pace would have required a much bigger team.

But the bottleneck didn't disappear; it moved. It is no longer in writing code. It is in three activities that remain human:

1. **specifying**: deciding what we want, with verifiable criteria;
2. **reviewing**: every PR is reviewed automatically, but every single one was merged by hand, by a human;
3. **validating on site**: a game is only finished once a team has played it.

One piece is still missing: a CI that builds and runs those 800 tests on every pull request. Today, review is automated; running the tests isn't yet.

My conclusion is simple. AI did not design Heroes Academy. It arrived on an architecture designed without it, and that architecture is precisely what made it effective: small classes, a documented MQTT contract, hardware described as data, injected time. An agent is an excellent developer for a well-partitioned system. On a monolith where everything depends on everything, as V1 was, I doubt it would have done better than I did.

---

> **The Heroes Academy series**
> 1. [Designing software for a physical game venue](./heroes-academy-1-v1.md): V1, MongoDB as a bus, the zombie process.
> 2. [The V2 rewrite for Canada](./heroes-academy-2-v2-rewrite.md): V2's architecture, principles and bugs.
