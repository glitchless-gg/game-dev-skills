---
name: unreal-performance
description: Find and fix hitches, frame drops and memory growth in Unreal Engine 5 games. Measure a packaged Development build with stat unit, stat game, Unreal Insights and the PSO hitch log, then fix the usual causes (pipeline states created at draw time, garbage collection from too many UObjects, map loads and level streaming on the game thread, synchronous asset loads, Blueprint Tick, draw calls and shadows) and prove the fix with the same measurement. Use when an Unreal game stutters, hitches the first time something appears, freezes on a level change, leaks actors or UObjects, or runs badly on the Steam Deck or lower-end PCs.
---

# Unreal Engine 5 performance

Start with the general loop in `game-performance` (budget, baseline, one change, measure again). This skill
is the Unreal part.

## Measure a packaged build

Play In Editor compiles shaders on first use and carries the editor's own cost, so PIE hitches are often not what
players see. Confirm everything in a **packaged Development build**.

- `stat unit`: Frame, Game (game thread), Draw (render thread), RHIT and GPU times. The largest one is the limit.
- `stat game`, `stat scenerendering`, `stat rhi` (draw calls), `stat gpu` or `ProfileGPU`.
- **Unreal Insights**: launch with `-trace=default` and open the trace to see which function took a long frame.
- **PSO hitches**: `-LogCmds="LogPSOHitching Verbose"` names the material and pass of each pipeline state created
  at draw time.
- **Memory**: `obj list class=<Class>` and `memreport -full` show which class keeps growing.

Report p50, p95, p99 and the worst frame in ms, from the same route each time.

## What the signs mean

| Sign | Likely cause |
| --- | --- |
| a hitch the first time a material, mesh or effect is drawn | a pipeline state (PSO) created on the spot |
| a hitch every so often that grows over a session | garbage collection with more and more UObjects |
| a freeze on a level change | `OpenLevel` loading the whole map on the game thread |
| a hitch walking into a new area | a streamed level added in one frame, or blocking on its load |
| a hitch on first use of an asset | a hard reference or `LoadSynchronous` on the game thread |
| Game time high every frame | Blueprint Tick doing loops or searches, too many actors ticking |
| Draw, RHIT or GPU high | draw calls, shadow-casting lights, translucency overdraw |

## Fixes

**PSOs.** Keep PSO precaching on (`r.PSOPrecaching 1`, the default) so components compile theirs while they load.
Ship a bundled PSO cache for what precaching misses (materials set at runtime, effects spawned later): play a
Development build with `-logPSO`, turn the recorded files into a cache with the `ShaderPipelineCacheTools`
commandlet and add it to the build (`r.ShaderPipelineCache.Enabled`). Spawn first-use effects once behind the
loading screen.

**Garbage collection.** Its cost grows with the number of UObjects. Pool actors and objects (hide and reuse them)
instead of `SpawnActor` or `NewObject` for every shot, spark or damage event. Keep short-lived data in
`USTRUCT`s or plain C++. Cluster what lives long (`gc.CreateGCClusters`, `gc.ActorClusteringEnabled`), and run
full collections where players don't see them (`ForceGarbageCollection` behind a loading screen).

**Leaks.** Anything a `UPROPERTY` references survives garbage collection: clear `TArray` and `TMap` properties that
still point at objects from an earlier level. `Destroy()` actors when they're done, or give short-lived ones a
lifespan (`SetLifeSpan`).

**Map loads and streaming.** Cover `OpenLevel` with a loading screen (the MoviePlayer), or stay in a persistent map
and stream levels with Level Streaming or World Partition. Request streamed levels before the player gets there,
never block on them (`bShouldBlockOnLoad`, `FlushLevelStreaming`), and spread registering their actors over frames
(`s.LevelStreamingComponentsRegistrationGranularity`, `s.AsyncLoadingTimeLimit`).

**Asset loads.** Load ahead with the Asset Manager or `FStreamableManager::RequestAsyncLoad` (Async Load Asset in
Blueprint) and use soft references for what a level may not need.

**Game thread.** Move hot Blueprint loops to C++, replace `GetAllActorsOfClass` and per-frame searches with cached
references, turn Tick off (or raise the tick interval) on actors that don't need it, and use timers and events.

**Rendering.** Instanced static meshes, merged actors, HLOD for distant geometry, Nanite for dense meshes, fewer
shadow-casting lights, less translucency.

**Steam Deck and lower-end PCs.** Add a quality tier: lower scalability groups
(`UGameUserSettings::SetOverallScalabilityLevel`, or `sg.ShadowQuality`, `sg.GlobalIlluminationQuality`,
`sg.PostProcessQuality` one by one), a lower `r.ScreenPercentage` with TSR upscaling, cheaper lighting than Lumen
(`r.DynamicGlobalIlluminationMethod`), and plain shadow maps (`r.Shadow.Virtual.Enable 0`).

## From real players

Insights shows one machine and one run. To see hitches with their cause (PSO, GC, map load, streaming) from every
player's session, by GPU, version and map, add [Glitchless](https://www.glitchless.gg) (open source, MIT;
a plugin for UE 5.8). Packaged builds can also measure themselves for a coding agent: `-glitchless-measure=10`
prints the numbers as one JSON line, and with Unreal MCP in the editor the Glitchless Toolset adds a `Measure`
tool. The `glitchless-setup` skill covers installing it.
