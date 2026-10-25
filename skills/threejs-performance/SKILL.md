---
name: threejs-performance
description: Find and fix frame drops, stutter and memory growth in Three.js and React Three Fiber games. Measure with renderer.info and frame-time percentiles, then fix the usual causes (draw calls from many meshes, objects never disposed, first-use shader compiles, shadow-casting lights, pixel ratio, per-frame allocations) and prove the fix with the same measurement. Use when a three.js or R3F game is laggy, gets slower as you play, hitches when something appears for the first time, runs under 60 fps, or is slow on phones.
---

# Three.js performance

Start with the general loop in `game-performance` (budget, baseline, one change, measure again). This skill
is the Three.js part: how to measure and what usually causes it.

## Measure

`renderer.info` is the first thing to read. Log it every few seconds while you play:

```js
setInterval(() => {
  const { render, memory, programs } = renderer.info;
  console.log({
    calls: render.calls,          // draw calls last frame
    triangles: render.triangles,
    geometries: memory.geometries, // live geometries on the GPU
    textures: memory.textures,
    programs: programs?.length,    // compiled shader programs
  });
}, 3000);
```

With several render passes per frame (post-processing, a minimap), set `renderer.info.autoReset = false` and call
`renderer.info.reset()` once per frame yourself, or `calls` only shows the last pass.

Then time frames, not fps: record `performance.now()` deltas in your loop and report p50, p95, p99 and the worst
frame over 10 to 30 seconds. Chrome's Performance panel shows which function a long frame spent its time in.
Spector.js captures one frame's draw calls.

What the numbers say:

| Sign | Likely cause |
| --- | --- |
| `calls` climbs the longer you play | meshes added and never removed (effects, pickups, bullets) |
| `geometries` or `textures` climbs on every level load | removed from the scene but never `dispose()`d |
| `programs` jumps when something first appears, with a hitch | a new shader compiled on that frame |
| `calls` high from the start (thousands) | one mesh per object where one instanced draw would do |
| fine on desktop, slow on phones | pixel ratio, shadows, overdraw, too many lights |

## Usual causes and fixes

**Many copies of the same thing.** One `Mesh` per coin, tree, bullet or particle is one draw call each (two with
shadows). Draw them with one `InstancedMesh` (or `THREE.Points` for particles) and update the instance matrices.
Share one geometry and one material between copies; never create them per object.

**Effects that never go away.** Fading a mesh to opacity 0 keeps it in the scene and in every draw. Remove it when
the effect ends and free what it owned:

```js
scene.remove(mesh);
mesh.geometry.dispose();
mesh.material.dispose();      // and each texture it alone used: material.map?.dispose()
```

Better still, pool them: hide and reuse a fixed set instead of creating new ones per effect.

**Level unloads that leak.** `scene.remove()` alone frees nothing on the GPU. Traverse what you unload and
dispose geometries, materials and textures that aren't shared, or `renderer.info.memory` keeps rising.

**First-use shader compiles.** Three.js compiles a material's program the first frame it's drawn, which is a
hitch you see once per material. Compile them during loading:

```js
await renderer.compileAsync(scene, camera); // with everything the level uses in the scene, even off-camera
```

Adding or removing a light changes every lit material's program and recompiles them all. Keep the number of
lights constant: move a small pool of lights where they're needed, or use emissive materials and glow sprites.

**Shadows.** Each shadow-casting light draws the scene again into its shadow map. Cast shadows from one light,
set `castShadow` only on objects that need it, keep `shadow.mapSize` modest (1024 to 2048), and tighten the
shadow camera around the play area.

**Pixel ratio and overdraw.** Phones report `devicePixelRatio` of 3, which is nine times the pixels of 1. Cap it:
`renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5))`. Big transparent layers, full-screen
post-processing and many overlapping particles cost per pixel; cut them first on mobile.

**Per-frame allocations.** `new Vector3()`, arrays and closures inside the animation loop make garbage, and
garbage collection shows up as periodic hitches. Allocate once outside the loop and reuse.

**Textures.** Large PNGs take GPU memory and upload time. Use power-of-two sizes no bigger than needed, and
KTX2 (Basis) compressed textures with `KTX2Loader`; compress meshes with Draco or Meshopt.

## React Three Fiber

- Allocations in `useFrame` run every frame: keep vectors and quaternions in `useMemo` or module scope.
- Don't `setState` in `useFrame`; mutate refs.
- Use drei's `<Instances>` / `<Instance>` or `<instancedMesh>` for repeated objects.
- `frameloop="demand"` for scenes that don't animate all the time; `<PerformanceMonitor>` to lower quality on
  slow devices.
- Unmounting a component disposes what R3F created for it, but not objects you created yourself with `new`.

## From real players

Your machine isn't your players'. To see frame times, hitches with their likely cause and `renderer.info`
counters from every session, by device and level, add [Glitchless](https://www.glitchless.gg) (open source):

```js
import * as THREE from 'three';
import { Glitchless, instrumentThree } from 'glitchless';

const glitchless = Glitchless.init({ service: 'my-game', version: '1.0.0', intake, clientToken });
instrumentThree(glitchless, { renderer, scene, camera, THREE });
```

`intake` and `clientToken` come from your own Glitchless server or from Glitchless Cloud. It also lets a coding
agent read those numbers over MCP and measure the running game in the browser (`glitchless_measure`,
`glitchless_inspect_scene`). The `glitchless-setup` skill covers the rest.
