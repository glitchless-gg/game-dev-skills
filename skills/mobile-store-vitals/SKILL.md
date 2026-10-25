---
name: mobile-store-vitals
description: Read what Google Play and the App Store measure about a mobile game (Android vitals for slow rendering, crash and ANR rates and the slowest devices and GPUs; Apple's hang rate, launch time, animation hitches and heaviest hangs) and what reviewers say about each version, then find the cause in the game and confirm the fix in the next version. Use when Play Console warns about bad behavior or slow rendering, App Store reviews say the game freezes, a version's ratings dropped, or the user asks how a mobile game is doing on real devices.
---

# Mobile store vitals and reviews

The stores measure every install, including the phones you don't own. Their numbers are coarse (players per day,
by version) but they decide visibility: Google Play lowers a game's reach when it crosses its bad-behavior
thresholds. Read them per version, next to your own measurements.

## Google Play: Android vitals

In Play Console, **Monitor and improve → Android vitals** (or the Play Developer Reporting API):

- **Slow rendering**: the share of users whose frames ran slower than 20 fps and 30 fps.
- **Crash rate** and **ANR rate** (the app not responding for 5 seconds): Google's bad-behavior thresholds are
  about 1.09% user-perceived crashes and 0.47% user-perceived ANRs overall, and 8% on any single device model.
- **By device and GPU**: which models and GPU families are worst. One GPU family at the top is usually a shader or
  driver problem, not your game logic.
- **The pre-launch report** runs your build on Firebase Test Lab phones. Keep those sessions out of your own
  player numbers.

ANRs in games are usually the main thread blocked at startup (loading everything before the first frame), on a
synchronous file or network call, or in a long loop.

## The App Store: Xcode Organizer and App Store Connect

In Xcode, **Window → Organizer → your app** (or the App Store Connect API's metrics):

- **Hang rate**: seconds hung per hour of use. Organizer's **Hangs** report lists the heaviest hang signatures with
  their stacks, for the code that froze.
- **Launch time** and **animation hitch rate**, at the median and 90th percentile, per version.
- **Crashes** with symbolicated stacks (upload dSYMs).

## Reviews by version

Both stores record the version a review was written on (Play also the device). Count the negative share about
performance (freezes, crashes, lag, stutter, battery, heat) per version, in every language, and note the devices
reviewers name.

## Find and confirm the cause

1. Pick the version where a vital or the review topic jumped, and the one before it.
2. Diff what changed between them: new effects, shaders, a renderer switch, an SDK update, bigger assets.
3. Reproduce on the device class that's struggling (a mid-range Android phone with that GPU; an older iPhone), with
   the `game-performance` loop and the engine skill (`godot-performance`, `unity-performance`, ...).
4. Ship the fix as a new version and compare that version's vitals and reviews with the old one's after a few days.
   Store vitals lag by a day or two.

## With Glitchless

[Glitchless Cloud](https://www.glitchless.gg/google-play.html) connects a Play listing (a service account's JSON
key) and an [App Store listing](https://www.glitchless.gg/app-store.html) (the Apple ID, plus an App Store Connect
key for Apple's measures), read only. It puts each store's vitals next to what Glitchless measured on the same
version, with a line wherever the two disagree and why (a store counts users per day, Glitchless counts sessions;
native crashes before the SDK starts never reach Glitchless). A coding agent reads them with
`glitchless_store_vitals` and `glitchless_store_reviews`, and `glitchless_triage` raises a finding when a version's
reviews turn against its performance. The Godot and Unity SDKs mark Play's pre-launch test phones as bots.

Ask your agent:

```
Android vitals say slow rendering doubled on our latest version. Which devices and GPUs are hit, what do reviews of
that version say, and what did Glitchless measure on it against the version before?
```
