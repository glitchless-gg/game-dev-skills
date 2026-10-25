---
name: phaser-performance
description: Find and fix frame drops, stutter, memory growth and frozen games in Phaser 3 and 4. Measure frame times and the counts that grow (game objects, physics bodies, particles, tweens, event listeners, textures), then fix the usual causes (objects never destroyed, a collider per object, handlers added on every scene restart, textures made mid-game, Graphics redrawn every frame, uncaught errors that stop the game loop) and prove the fix with the same measurement. Use when a Phaser game lags, gets slower after restarts, hitches when things spawn, freezes, or runs badly on phones.
---

# Phaser performance

Start with the general loop in `game-performance` (budget, baseline, one change, measure again). This skill
is the Phaser part.

## Measure

Log the counts that tend to grow, every few seconds while you play and after each scene restart:

```js
this.time.addEvent({ delay: 3000, loop: true, callback: () => {
  const scenes = this.game.scene.getScenes(true);
  console.log({
    fps: Math.round(this.game.loop.actualFps),
    objects: scenes.reduce((n, s) => n + s.children.length, 0),
    bodies: this.physics?.world?.bodies?.size,             // Arcade Physics
    tweens: this.tweens.getTweens().length,
    listeners: this.events.eventNames().reduce((n, e) => n + this.events.listenerCount(e), 0),
    textures: Object.keys(this.textures.list).length,
  });
}});
```

Record frame deltas (`this.game.loop.delta`) and report p50, p95, p99 and the worst frame. Chrome's Performance
panel shows whether a long frame was in your `update()` or in rendering.

## What the signs mean

| Sign | Likely cause |
| --- | --- |
| objects or bodies climb as you play | sprites that fade out or leave the screen but are never destroyed |
| listeners climb on every restart | `this.events.on()` in `create()` without removing it on shutdown |
| a hitch when something spawns | many objects created in one step |
| a hitch the first time an effect shows | a texture generated mid-game and uploaded on that frame |
| slow every frame, many texture switches | sprites spread over many textures instead of an atlas |
| the game freezes for good | an uncaught error in `update()`, a physics callback or an event handler |

## Fixes

**Recycle instead of create.** Create objects once and reuse them through a Group:

```js
this.bullets = this.physics.add.group({ maxSize: 50 });
const b = this.bullets.get(x, y, 'bullet');   // a dead one if there is one
if (b) { b.setActive(true).setVisible(true); }
// when it's done:
this.bullets.killAndHide(b); b.body.enable = false;
```

`destroy()` what you're really done with. Faded or off-screen bodies keep simulating until destroyed or disabled
(`disableBody(true, true)`).

**One collider per group**, not per object: `this.physics.add.collider(player, enemies)`.

**Scene restarts.** Scene events outlive a restart, so handlers added in `create()` pile up. Remove them on
shutdown, or use `once()`:

```js
this.events.once('shutdown', () => this.game.events.off('score', this.onScore, this));
```

**Textures.** Make generated textures (`generateTexture`, render textures, canvases) once in the preloader and
reuse the key; every new texture is uploaded to the GPU the frame it's made and stays in memory until removed.
Pack images into atlases to cut texture switches.

**Graphics.** Don't clear and redraw a `Graphics` object every frame; draw it once, or bake it to a texture.

**Particles and tweens.** Lower emitter quantity and lifespan, `stop()` emitters off screen, tween a container or a
group instead of one tween per object, and stop looping tweens you no longer need.

**Garbage collection.** Avoid creating objects, arrays and closures every frame in `update()`.

**Uncaught errors stop the game.** In Phaser an error thrown inside `update()`, a physics callback or an event
handler stops the game loop for good: the player sees a frozen screen. Guard lookups that can return null
(`group.get()`, `getFirstDead()`, a destroyed object still referenced from a timer or tween), and apply game state
(score, pickups) before anything that can throw.

**Phones.** A smaller game size with `Scale.FIT`, resolution 1, fewer preFX/postFX filters and lights, half the
particles.

## From real players

To see frame times, hitches with their cause, these counts and errors (including the ones that froze the game)
from every session, add [Glitchless](https://www.glitchless.gg) (open source, MIT):

```js
import { Glitchless } from 'glitchless';
import { instrumentPhaser } from 'glitchless/phaser';

const glitchless = Glitchless.init({ service: 'my-game', version: '1.0.0', intake, clientToken });
instrumentPhaser(glitchless, { game, Phaser });
```

The adapter sends game objects, physics bodies, particles, tweens and listeners for you, and a coding agent can read
it all over MCP. The `glitchless-setup` skill covers the rest.
