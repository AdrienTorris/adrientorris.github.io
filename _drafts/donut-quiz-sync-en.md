---
title: "Donut Quiz: syncing a head office and its venues without silently losing anything"
description: "A watermark that followed the wrong clock, properties vanishing behind a 200 OK, an enrolment that 'succeeded' without doing anything: lessons learned from the content sync between the head office and the Donut Quiz rooms."
lang: en
tags: [dotnet, synchronization, mongodb, hangfire, system-text-json, architecture, lessons-learned]
---

# Donut Quiz: syncing a head office and its venues without silently losing anything

> This post follows on from [Donut Quiz: behind the scenes of a connected quiz room](./donut-quiz-article-en.md), which covers the room, its architecture and the software rewrite. Here I focus on a single piece of the puzzle: the sync between the head office and the venues.

A Donut Quiz room has to keep playing even if the venue's internet connection drops. So each venue has its own MongoDB database and its own media volume (13 to 19 GB of images, videos and sounds). Content (questions, themes, special events, slides, award cards, texts, game settings…) is written at the head office and flows down to the venues. In the other direction, played games, customers and the history of questions asked to each customer flow up. That history lets the game generator avoid asking a returning customer the same question twice.

On paper, this is a textbook problem. In practice, it's the part of the system where defects are the sneakiest, because a sync that loses data doesn't crash: it answers 200, reports success, and the gap only shows up weeks later, when a game master wonders why a special event is missing.

> **TL;DR**
> - The watermark followed the venue's clock, while the data is timestamped by the head office. It is now derived from what was actually written.
> - System.Text.Json, with `IgnoreReadOnlyProperties`, silently dropped every property without a public setter. A startup guard now refuses to launch the application if a property doesn't make it across.
> - A venue can see what it's missing and trigger a sync on demand. The head office can see the state of the whole fleet.
> - Content can belong to a franchisee, with strict isolation between venues.
> - The head office never connects to a venue: everything starts from the venue, over outbound HTTPS.

---

## How it works

In the first version of the software, the sync was the last component to arrive. It ran at night, between 1 a.m. and 5 a.m., except for games, which synced every five minutes. The principle hasn't changed, but almost everything else has.

- **A dedicated service.** On the venue side, `ContentSynchronization.AgenceApp` runs Hangfire jobs, stored in MongoDB. Fifteen entities travel: all of them flow down, and three also flow up (games, customers, question history).
- **A staggered schedule.** Content syncs at night. Each venue picks a random start time within a five-hour window beginning at 10 p.m., which spreads the load on the head office. Games and customers sync every five minutes, because a booking made online has to reach the room quickly.
- **One watermark per entity.** Each entity has its own watermark (`Since`). The venue calls `POST /siege/api/sync/{Entity}/pull` to get everything that changed since that point (`LastUpdated >= Since`). The head office returns the full entities along with their media links, and the venue writes them to its database, then downloads the media into the shared volume.
- **Purging deleted content.** A `check-dead` route lets the venue ask which of its items no longer exist at the head office, so it can purge them.
- **Token-based enrolment.** To enrol, a venue exchanges a single-use generic token for its own venue token, then registers its code with the head office.
- **One worker per queue**: one ffmpeg conversion at a time, and never two pulls of the same entity in parallel.

Nothing revolutionary. As always, the devil is in what happens when things go wrong… without saying so.

---

## Two clocks, one watermark

At the end of each run, the watermark was set to the venue's local time (`DateTime.Now`). But the documents' `LastUpdated` is set by the **head office** clock.

A venue clock running a few minutes ahead is enough to create a permanent gap: a document changed at the head office during that window will never pass the `LastUpdated >= Since` filter again.

To top it off, an item that arrived without data was counted as "applied". So a run could move the watermark forward without writing a single row. And the field meant to record the last application never changed after the tracking document was created: a run that applied nothing left exactly the same trace as a run that applied everything.

The watermark is now derived from what was **actually written**:

```csharp
// Before: the venue's local time, whatever happened
contentSynchronizationInfo.Since = executionDate; // DateTime.Now, venue side

// After (in principle): the highest LastUpdated actually applied, stamped by the head office,
// capped if implausible, pulled back to the oldest failed item,
// and left unchanged if nothing was written.
```

An item without data no longer counts as applied, and the tracking document records a real last-application date. The clock itself is injected (`ISynchronizationClock`), so clock-drift scenarios can be tested without waiting for a venue to produce them.

---

## A 200 OK proves nothing

Both ends serialize with System.Text.Json and `IgnoreReadOnlyProperties = true`. With that setting, a property without a public setter is **neither written by the sender nor read by the receiver**. No exception, no trace, and a 200 in response.

Pushing games up stacked several problems:

- it went through a projection DTO that declared only 20 of a game's 35 properties;
- that DTO's team list was never filled in. Since the head office replaces the whole document on every push, it received **games without teams, and therefore without scores or rankings**;
- finished games had one more problem: their awards were stored in a dictionary that MongoDB refused to persist in its default representation, so **the whole game** was rejected. Precisely the games the head office needs most.

The "obvious" fix, removing `IgnoreReadOnlyProperties`, would have broken the pull: some entities expose computed properties typed as interfaces, which the deserializer can't instantiate. The flag was actually hiding a domain model that had never been designed to be serialized.

### A guard rather than a list

So I went property by property with `[JsonInclude]`. But a list of properties to watch ages badly: the next property added with a private setter will bring the defect back without anyone noticing.

So at startup, a guard uses reflection to inspect each of the fifteen entities, in both directions. If a property doesn't make it across, the application refuses to start. The only properties that escape it are computed ones, exempted automatically since they have no setter, and two properties exempted by name, with the reason written down.

The error message names the type, the property, the direction and the cause, and never just a count:

```csharp
/// "3 properties lost" would survive a regression on a DIFFERENT property:
/// the count doesn't move, the message stays true, and the loss quietly changes victims.
public override string ToString()
    => $"{Entity.Name}.{Property} does not travel in direction {Direction}: {Cause}";
```

An inventory of what travels, property by property and in each direction, is also versioned in the repository. A test checks that it covers all fifteen entities in both directions: deleting a section makes it fail.

### A played game is a fact

Once game rounds made it across, a new trap appeared. The question bank contained incomplete questions, and model validation would have rejected outright any game that included one. But a played game is a fact: you don't reject it.

So when a game is pushed, and for that action only, model validation is lifted. The game is written as is, and each missing field produces a warning naming the game, the round and the property. The question bank gets cleaned up by observation, not by blocking. Every other head-office endpoint keeps its validation.

### Everyday silences

While inspecting the sync, I found a job that had been calling, every five minutes, a route the head office didn't serve: no controller existed for that entity. It got a 404, applied nothing… and nothing reported it.

Finally, the venue stays in charge of what it plays. A game in progress is never overwritten by the sync, and a local copy at least as recent as the head office's wins.

---

## See what's missing, and pull it without waiting for the night

When a game master notices that a new special event is missing, waiting for the nightly sync isn't an answer. But first, you have to know what's missing: until then, the only way to count missing questions was to download every question.

- **The head office can count.** An inventory endpoint returns, per entity, how many items the venue doesn't have yet, without downloading their content.
- **The venue sees what it's missing.** Its admin site shows those counters, entity by entity, with a single button that triggers the pull right away.
- **Clicking is safe.** Repeated requests are coalesced per entity: a click while a pull is queued or running doesn't start a second one. And no pull starts while a game is running.
- **The head office sees the whole fleet.** Each venue reports its watermark. A fleet view shows which venues are up to date and which are lagging, along with how fresh that information is.

The admin site and the scheduler are two separate processes. So the admin button calls an HTTP endpoint on the scheduler, restricted to the local origin, while role checks stay entirely in the admin site.

### An enrolment that no longer succeeds silently

A venue's enrolment could "succeed" silently through four different exits, without a single log line. And a token being present was taken as proof that it was valid: an expired token, a revoked one, or one issued by another head office would do just as well.

Only the head office can judge a token. It now checks it and says why it refuses, while the venue logs and displays the reason. A green screen no longer means "a string happens to be present in a file".

---

## Content for each franchisee

The latest change: franchisees now create content at the head office for their own venue only.

- **An optional owner.** A question or a special event has an owner. No owner means "global", which avoided any migration of existing data.
- **A sync filtered by owner.** The pull, the counters, the media links and the detection of deleted content all take the owner into account, with strict isolation: a venue never receives another venue's content.
- **No central approval.** A venue's content goes to its room as soon as it's enabled, and only to the owning venue.
- **"Putting it in the pot".** Venue content that deserves to benefit the whole network can be copied as global content.

---

## No inbound connection to a venue

Even to retrieve the game history of the older venues, the rule stays the same: **the head office never connects to a venue.** No inbound access, no SSH key.

```
VENUE (read-only)                               HEAD OFFICE
  mongodump --archive --gzip                    PUT /…/ingest/<venue>/<collection>
        │  (streamed, no temporary file)          └─> archive dropped
        └── curl -T -  ──── outbound HTTPS ───>   restored into a throwaway database
                                                  then merged into the main database
```

The ingestion route is token-authenticated and **inert unless configured**: without a configured token, it answers 404. It can therefore be deployed anywhere safely, since the configuration, and only the configuration, turns it on. On the venue side, everything is read-only: no service installed, no temporary file, no write to MongoDB, and the operation can be interrupted at any time.

---

## What I take away from this

- **A 200 OK proves nothing.** Judge a sync by what was actually written.
- **A clock is only valid for the machine that read it.** Derive a watermark from the data you received, never from local time.
- **A guard beats a list.** What must stay true should be checked at startup, by construction, rather than in a list nobody will keep up to date.
- **Name things, don't count them.** A message that names the lost property can't stay true after a regression somewhere else.
- **The room stays in charge of what it plays.** A game in progress is never overwritten.
- **Never connect inbound to a venue.** Everything starts from the venue, over outbound HTTPS.

For the rest of the story (web-based desks, MQTT, sound, lighting and deploying to Raspberry Pis), see the main post: [Donut Quiz: behind the scenes of a connected quiz room](./donut-quiz-article-en.md).
