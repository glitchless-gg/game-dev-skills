---
name: game-analytics
description: Decide what a game should measure and wire it up, in any engine. Turn the developer's real questions (which mode do people play, where do they quit, does the tutorial work, does the paywall convert, which level is too hard) into a small set of events with attributes, level start and end, funnels and a dashboard in the game's own terms, without tracking anything about the player. Use when the user wants analytics, metrics, KPIs, a game dashboard, retention or funnel numbers, or asks "how many players…", "which…", "where do players drop off".
---

# Game analytics: measure what this game's developer wants to know

Every game is different. A puzzle game cares about solve times and hints, a roguelite about how far runs get and
which upgrades win, a mobile game about opens, modes, the paywall and purchases. Don't start from a template of
"standard game events". Start from questions.

## 1. Get the questions

Ask the developer (or read the design doc, store page and code) for the five to ten things they'd check every
week. Good questions name a decision:

- "Which mode do people actually play?" decides where to spend the next month.
- "Where do new players stop?" decides what to fix in the first ten minutes.
- "Does the tutorial help, or do people skip it and quit?"
- "Which level, stop or boss ends most runs?"
- "How many people who see the unlock screen buy, and where did they open it from?"
- "Did the last update make runs longer or shorter?"

## 2. Turn each question into events

- **One event per moment**, named for what happened: `play`, `run_end`, `tutorial`, `paywall`, `purchase`,
  `boon_picked`. Keep names short and stable; renaming an event splits its history.
- **A few attributes** that answer the "which": `{ mode: 'endless' }`, `{ how: 'lost', depth: 4 }`,
  `{ where: 'settings' }`. Strings, numbers or booleans. Low cardinality: a mode, not a timestamp.
- **Levels, stages and runs** get start, complete and fail with a cause, so completion rates and walls fall out
  of them.
- **Steps of a flow** become a funnel: `paywall` → `buy` (tapped) → `buy` (done). Send a step attribute rather
  than inventing event names per step.
- **Purchases**: one `purchase` event when the store confirms it, with `amount` (4.99, not 499), `currency` and
  `item`. Mark test purchases (TestFlight, sandbox) so they never count as revenue.
- **Version** on everything, so you can compare before and after an update.

Never send anything about the player: no names, emails, account ids, IPs, receipts or free text they typed.
Count sessions and anonymous players.

## 3. Check the data before trusting it

Play through once and confirm each event arrives with the attributes you expect. Then check the totals add up:
`purchase` events against the store's dashboard, `run_end` against `play`. A metric you haven't checked will be
quoted in a meeting anyway.

## 4. Design the dashboard around the questions

One page per question area, in the game's words: "Modes and worlds", "Where players stop", "Free to full",
"Expedition". On each page, lead with the number that answers the question (a share or a rate, with its count),
then the breakdown, then the trend over days and versions. Leave out panels nobody would act on.

## With Glitchless

[Glitchless](https://www.glitchless.gg) (open source, MIT) takes these events from every engine and builds each
game its own dashboard, next to frame times, crashes and where players quit:

| Engine | Track an event |
| --- | --- |
| Three.js, Phaser, Babylon.js, PlayCanvas | `glitchless.track('play', { mode: 'endless' })` |
| Godot | `Glitchless.track("play", "endless")` or a dictionary |
| Unity | `Glitchless.Track("play", "endless");` or a `Dictionary<string, object>` |
| Unreal | `FGlitchless::Track(TEXT("play"), TEXT("endless"));` or Track Event in Blueprint |
| GameMaker | `glitchless_track("play", "endless");` or a struct |

A coding agent can read the events and funnels over MCP (`glitchless_events`) and save the dashboard pages it
designed with `glitchless_set_dashboard`. The `glitchless-dashboard` skill in this repo walks through that, and
`glitchless-setup` installs the SDK.
