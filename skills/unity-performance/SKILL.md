---
name: unity-performance
description: Find and fix frame drops, GC hitches, slow scene loads and memory growth in Unity games. Measure a development build with the Profiler, Frame Debugger and Memory Profiler, then fix the usual causes (per-frame allocations and garbage collection, Instantiate and Destroy bursts, first-use shader variants, synchronous scene loads, physics, draw calls and SetPass calls) and prove the fix with the same measurement. Use when a Unity game stutters, lags, freezes on load, leaks memory, or runs badly on phones or the Steam Deck.
---

# Unity performance

Start with the general loop in `game-performance` (budget, baseline, one change, measure again). This skill
is the Unity part.

## Measure the build, not the editor

The editor adds its own cost (the Scene view, the Inspector, editor-only checks), so a game at 10 fps in the
editor can be fine in a build. Measure a **Development Build** with **Autoconnect Profiler** on:

- **Profiler**, CPU module: Timeline view for one slow frame, Hierarchy sorted by self time to find the function.
  Watch **GC.Alloc** per frame: anything above zero in steady gameplay is garbage you'll pay for later.
  Deep Profile adds a lot of overhead; use it briefly, or add `ProfilerMarker`s around your own code.
- **Rendering module** and the **Frame Debugger**: batches, SetPass calls and what each draw is.
- **Memory Profiler** package: take a snapshot before and after a level change and compare.
- **Profile Analyzer** package: compare two captures (before and after your fix) frame by frame.

Report p50, p95, p99 and the worst frame in ms, not average fps.

## What the signs mean

| Sign | Likely cause |
| --- | --- |
| a hitch every few seconds, `GC.Collect` in the frame | per-frame allocations |
| a hitch when an effect or material first appears | a shader variant compiled on first use |
| a hitch on spawning | `Instantiate()` of many objects in one frame |
| a freeze on a level change | a synchronous scene load, heavy `Awake()`/`Start()` |
| long `Update()` frames | big loops, `GetComponent()` / `Find()` every frame |
| long physics steps | many rigidbodies, complex colliders, the catch-up spiral |
| memory climbs per level | runtime meshes, materials or textures never destroyed |

## Fixes

**Garbage collection.** Cut per-frame allocations: new lists and arrays, LINQ, string building and
concatenation, boxing, `GetComponents()` without a list to fill, closures in `Update()`. Cache what `Update()`
needs. Turn on **Incremental GC** in Player Settings so collections spread over frames.

**Spawning.** Pool and reuse objects instead of `Instantiate()` and `Destroy()` per bullet or effect:

```csharp
using UnityEngine.Pool;
var pool = new ObjectPool<GameObject>(
    () => Instantiate(prefab),
    go => go.SetActive(true),
    go => go.SetActive(false),
    go => Destroy(go),
    maxSize: 200);
```

Draw many copies with one particle system or GPU instancing instead of an object each.

**Shader variants.** On DirectX 11 and OpenGL, warm them behind the loading screen with a
`ShaderVariantCollection` (`WarmUp()`), and strip variants you don't use. On DirectX 12, Vulkan and Metal the
driver can still stall on a new vertex layout or render target: Unity 6's `GraphicsStateCollection` (experimental)
records the pipeline states a run uses (`BeginTrace()`/`EndTrace()`) and warms them up at load.

**Scene loads.** `SceneManager.LoadSceneAsync()` with `allowSceneActivation` for a loading screen, or Addressables.
Keep `Awake()`/`Start()` of the new scene light. `Resources.Load` of a large asset blocks the main thread.

**Scripts.** Spread big loops over several frames (a coroutine, or the Jobs system with Burst), and cache
`GetComponent()` and `Find()` results.

**Physics.** Fewer active rigidbodies and colliders, simpler collider shapes, a larger Fixed Timestep, and a
Maximum Allowed Timestep that stops the physics catch-up spiral.

**Rendering.** Cut draw calls and SetPass calls: the SRP Batcher (URP/HDRP), static and dynamic batching, GPU
instancing, fewer real-time shadow-casting lights.

**Memory.** Destroy runtime-created meshes, materials and textures when you're done (`renderer.material` makes a
copy every time you read it; use `sharedMaterial` or a `MaterialPropertyBlock` for per-object tweaks).

**Weak devices.** Add a quality tier: a lower `QualitySettings` level, a lower render scale (`renderScale` on the
URP asset), no shadows or post-processing, half the particles. Use ASTC textures on phones.

## From real players

The Profiler shows your machine. To see frame times, GC and other hitch causes, errors with their device, and
where players quit, from every session, add [Glitchless](https://www.glitchless.gg) (open source, MIT; a UPM
package for Unity 6.3+). Builds can also measure themselves for a coding agent:
`--glitchless-measure=10 --glitchless-out=measure.json` writes the same numbers to a file. The `glitchless-setup`
skill covers installing it.
