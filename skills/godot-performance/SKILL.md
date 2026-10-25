---
name: godot-performance
description: Find and fix frame drops, hitches, slow starts and memory growth in Godot 4 games, on desktop and on phones. Measure with the Profiler, Visual Profiler and Monitors (or Performance.get_monitor in a build), then fix the usual causes (first-use shader and pipeline compiles, node bursts and orphan nodes, scene loads on the main thread, physics, draw calls, the Compatibility renderer's GPU limits on Android) and prove the fix with the same measurement. Use when a Godot game stutters, lags, freezes on load, leaks memory, or runs badly on mobile, web or the Steam Deck.
---

# Godot 4 performance

Start with the general loop in `game-performance` (budget, baseline, one change, measure again). This skill
is the Godot part.

## Measure

**In the editor:** run the game, open the Debugger panel.

- **Profiler**: script time per function. Start it, play the slow part, stop it, sort by self time.
- **Visual Profiler**: CPU and GPU time per rendering step (only on Forward+ and Mobile; the Compatibility
  renderer doesn't time the GPU).
- **Monitors**: FPS, process and physics time, draw calls, objects, nodes and **orphan nodes**.

**In an exported build**, which is what players run, read the same monitors yourself:

```gdscript
func _ready() -> void:
	var t := Timer.new()
	t.wait_time = 3.0
	t.autostart = true
	t.timeout.connect(func():
		print({
			"fps": Engine.get_frames_per_second(),
			"process_ms": Performance.get_monitor(Performance.TIME_PROCESS) * 1000.0,
			"draw_calls": Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME),
			"nodes": Performance.get_monitor(Performance.OBJECT_NODE_COUNT),
			"orphans": Performance.get_monitor(Performance.OBJECT_ORPHAN_NODE_COUNT),
		}))
	add_child(t)
```

FPS averages hide hitches. Record each frame's delta in `_process` and report p50, p95, p99 and the worst frame.

## What the signs mean

| Sign | Likely cause |
| --- | --- |
| a hitch the first time an effect, enemy or material appears | pipeline (shader) compile on first draw |
| a hitch on spawning a wave, burst or explosion | many nodes `instantiate()`d and `add_child()`ed in one frame |
| orphan nodes climb on every level change | nodes removed with `remove_child()` but never freed |
| a freeze of half a second or more on a level change | the next scene loaded on the main thread |
| process time high every frame | a big loop, sort or search in `_process`, or `get_node()` every frame |
| physics time high | too many active bodies, Area overlaps, complex shapes |
| fine on desktop, bad on Android | the Compatibility renderer's limits, overdraw, shadows (see Mobile) |

## Fixes

**Shader and pipeline compiles.** Draw every material, mesh and particle effect a level uses once behind the
loading screen (off-camera works) so it compiles before play, and don't create new material variants mid-level.
On Forward+ and Mobile, Godot 4.4+ ubershaders cover most of the rest. In Godot 4.5+, turn on the **Shader
Baker** in the export preset so the first frame doesn't wait on compiles.

**Node bursts.** Pool and reuse nodes instead of calling `instantiate()` and `add_child()` in a burst. Draw many
copies with one `MultiMeshInstance2D`/`MultiMeshInstance3D` or a `GPUParticles2D`/`3D` node instead of a node each.

**Orphan nodes.** `remove_child()` takes a node out of the tree but doesn't free it. Call `queue_free()` on what
you remove (or return it to a pool), and drop references to removed nodes from arrays, dictionaries and
autoloads. `Node.print_orphan_nodes()` in a debug build lists them.

**Scene loads.** Load the next scene in the background and switch when it's ready:

```gdscript
ResourceLoader.load_threaded_request(path)
# each frame, behind a loading screen:
if ResourceLoader.load_threaded_get_status(path) == ResourceLoader.THREAD_LOAD_LOADED:
	get_tree().change_scene_to_packed(ResourceLoader.load_threaded_get(path))
```

**Scripts.** Spread big loops, sorts and searches over several frames (or a `WorkerThreadPool` task), cache
`get_node()` and `$Path` lookups in `@onready` variables, and turn off `_process` on nodes that don't need it
(`set_process(false)`).

**Physics.** Fewer active bodies and `Area` overlaps, simple collision shapes, and `monitoring = false` on areas
that don't need it.

**Rendering.** Cut draw calls and shadow-casting lights; batch repeated meshes with MultiMesh.

**Slow start.** Keep the first scene small, `preload()` only what the first screen needs, load the rest with
`load_threaded_request()`, and use VRAM-compressed textures.

## Mobile and the Compatibility renderer

The Compatibility renderer (OpenGL ES 3 / WebGL 2) is the default for many mobile and web exports, and it has the
tightest limits.

- **Shaders that fail on some phones.** Errors like `Fragment shader active uniforms exceed
  GL_MAX_FRAGMENT_UNIFORM_VECTORS` come from the GPU or driver, not your scripts. The game usually keeps running,
  with the effect missing, so nothing crashes and nobody tells you. Find which devices send it, then simplify the
  shader or material it names (fewer uniforms, uniform arrays, lights or texture slots), or give those devices a
  simpler one.
- **GPU time isn't measured** on Compatibility, so GPU-bound frames show up as unexplained hitches. If they bunch
  on one level or effect, cut overdraw, full-screen effects, transparent layers and shadow-casting lights there.
- **Add a quality tier** for weak devices: a lower `Viewport.scaling_3d_scale` (around 0.7), no shadows, glow or
  SSAO in the `WorldEnvironment`, no MSAA, half the particles.

## From real players

Editor runs and your own phone miss what only some players hit, like the shader failure above. To get frame
times, hitches with their likely cause, errors with their device, and the monitors above from every session,
add [Glitchless](https://www.glitchless.gg) (open source, MIT; Godot 4.5+ addon). Its builds can also measure
themselves for an agent: run the game with `--glitchless-measure=10` and it prints one JSON line of the same
numbers. The `glitchless-setup` skill covers installing it.
