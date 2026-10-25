---
name: game-performance
description: Find and fix why a game stutters, lags or drops frames, in any engine (Three.js, Phaser, Babylon.js, PlayCanvas, Unity, Godot, Unreal, GameMaker). Measure first (frame-time percentiles against the frame budget, hitches, what was happening when they hit), find the cause, change one thing, and prove the fix with the same measurement. Use when the user says their game is laggy, stutters, hitches, freezes, drops to 30 fps, runs badly on phones or a Steam Deck, or "got slower".
---

# Game performance: measure, find the cause, prove the fix

Never fix performance from a guess. Agents (and people) who skip the measurement change the wrong thing, then
report a win nobody measured. Get numbers, change one thing, get the same numbers again.

## 1. Know the budget

| Display | Frame budget |
| --- | --- |
| 30 Hz | 33.3 ms |
| 60 Hz | 16.7 ms |
| 90 Hz (VR) | 11.1 ms |
| 120 Hz | 8.3 ms |

Average fps hides stutter. A game can average 58 fps and still hitch every few seconds. Report:

- **p50** (the typical frame), **p95** and **p99** (the bad frames), and the **worst frame**, all in milliseconds;
- **hitches**: frames over about twice the budget, counted per minute;
- the counts that explain them: draw calls, triangles, live objects or nodes, memory.

## 2. Tell the two problems apart

| What you see | What it is | Usual causes |
| --- | --- | --- |
| p50 over budget | Too much work every frame | draw calls, overdraw and transparency, shadow-casting lights, physics bodies, per-frame scripts, post-processing, resolution too high for the GPU |
| p50 fine, p95/p99 or worst frame bad | One-off spikes (hitches) | first-use shader or pipeline compiles, garbage collection, loading an asset mid-play, spawning many objects in one frame, texture uploads, a scene or level change |
| Gets worse the longer you play | Something leaks | objects removed from the scene but never freed, effects that fade out but stay alive, listeners added on every restart, per-tile geometry or materials never disposed |
| Fine on your machine, bad for players | A device or build difference | phones and low-end GPUs, a different renderer (WebGL vs WebGPU, Godot Compatibility vs Forward+), thermal throttling, the editor vs a release build |

## 3. The loop

1. **Reproduce on a fixed path.** Same level, same actions, same seed, same device. Autoplay or a scripted
   route beats playing by hand.
2. **Baseline.** Measure for 10 to 30 seconds. Write down p50, p95, p99, worst frame, hitches per minute,
   draw calls and object counts. Save it.
3. **Find what correlates.** Which counter climbs with the bad frames? What happened just before each hitch
   (a pickup, an explosion, a level start, a menu opening)? That event's code is where to look.
4. **Change one thing**, the root cause, not a symptom. Don't lower quality settings to hide a leak.
5. **Measure again** exactly as in step 2 and show a before-and-after table.
6. **Check nothing else broke.** Replay the same route and compare more than frame time: errors, game
   state, whether the level can still be finished. An agent that "fixed the stutter" by deleting the particle
   system has not fixed it.

## 4. Measure where the players are

Profile a release or development build, not only the editor: the Unity editor adds its own cost, and Unreal's
Play In Editor compiles shaders on first use. Test on the slowest device you support. A phone that is fine for
one minute can throttle after five.

Lab runs still miss what only some players hit: a shader that fails on one GPU family, a leak that shows after
40 minutes, a level that only stutters on 60 Hz Android phones. For those you need numbers from real sessions.
[Glitchless](https://www.glitchless.gg) (open source, MIT) records frame-time percentiles, hitches with their
likely cause, and the counters above from every session, split by device, version and level, and gives
coding agents that data over MCP (`glitchless_triage`, `glitchless_performance`). The `glitchless-setup`
skill in this repo adds it to a game.

## 5. Engine guides

Each engine has its own tools and its own usual suspects. Use the matching skill for the details:

- Three.js and React Three Fiber: `threejs-performance`
- Godot 4: `godot-performance`
- Unity: `unity-performance`
- Unreal Engine 5: `unreal-performance`
- Phaser 3 and 4: `phaser-performance`

## Report

End with the numbers, not adjectives. For example:

```
level-2, 30 s, same route, same device
                 before    after
p50              14.1 ms   8.9 ms
p95              51.0 ms   16.2 ms
worst frame      212 ms    31 ms
hitches/min      9.0       0.3
draw calls       4,652     286
cause: every pickup left 800 confetti meshes in the scene; now one reused InstancedMesh
```
