---
name: game-crash-triage
description: Find and fix the errors and crashes players hit in a shipped game (Unity, Godot, Unreal, Phaser, Three.js, Babylon.js, PlayCanvas, GameMaker). Group reports by fingerprint, rank by players affected, read the stack and the events just before it, reproduce, fix the cause, and confirm the fix in the next version. Covers each engine's usual culprits (freed nodes, destroyed objects, None references, unchecked lookups, GPU-specific shader failures) and errors that never crash but silently lose game state. Use when the user has crash reports, error logs, a bug players reported, a game that freezes, or "it works on my machine".
---

# Game crash and error triage

Most errors players hit don't crash anything. The engine skips the rest of a function and carries on, so a
pickup never adds its score, a menu never opens, or an effect never shows, and nobody writes in. Treat errors as
seriously as crashes.

## 1. Collect and group

Group reports by **fingerprint** (the error message with numbers stripped, plus the top stack frame), not by
occurrence. One loop can log the same error 500 times in one session. Rank each group by **sessions or players
affected**, then by whether it's fatal, then by count.

For each group, get:

- the message, the stack, and the file and line (minified web builds need source maps to be readable);
- the **events just before it** (breadcrumbs): level start, pickup, menu opened, purchase;
- the **version** it first appeared in, and whether it's still happening in the latest one;
- the **devices**: OS, GPU, renderer, screen. Errors on one GPU family are a device problem, not your logic.

## 2. Reproduce

Use the breadcrumbs as the script: load that level, do what the player did, on a device like theirs. If it only
happens in release builds, test a release build (several engines skip or swallow script errors there).

## 3. Fix by engine

**Godot.** Usually a node that is null or already freed (a `get_node()` path that no longer exists, a node freed
earlier: check `is_instance_valid()`), a node used before it's in the tree (`is_inside_tree()`), or a missing
dictionary key. Release exports skip most script errors silently, and a call on a typed null crashes the game.
Renderer errors such as `Program linking failed` or `exceed GL_MAX_FRAGMENT_UNIFORM_VECTORS` come from a GPU or
driver that can't build a shader: find which devices send it, then simplify that shader or give them a simpler one.

**Unity.** A reference that is missing or destroyed (`GetComponent()` or `Find()` returned null, an Inspector field
left empty, an object `Destroy()`ed earlier: `MissingReferenceException`), or a dictionary or array lookup. When an
exception is thrown, the rest of that `Update()` or coroutine is skipped every time.

**Unreal.** In Blueprint, a reference that is None (an actor destroyed earlier, a failed Cast, a variable never set):
add an Is Valid node. In C++, a UObject in a raw pointer without `UPROPERTY()` that garbage collection freed, an
actor pending kill, an index out of range, or a `check()` a player can reach. Keep UObjects in `UPROPERTY()` or
`TWeakObjectPtr`, test with `IsValid()`, and turn reachable `check()`s into `ensure()`s with a fallback. Package
with debug files (`.pdb` on Windows, `.dSYM` on Mac) so crash stacks have lines.

**Phaser.** An unchecked lookup (`group.get()` or `getFirstDead()` returning null, a destroyed object still used by a
timer, tween or physics callback). An uncaught error in `update()`, a physics callback or an event handler stops
the game loop for good: a frozen screen.

**Three.js, Babylon.js, PlayCanvas.** An unchecked `.find()` or pool lookup returning `undefined`, an object used
after `dispose()`, an asset that failed to load. The browser keeps the loop running, so the game looks fine while
state is wrong.

**GameMaker.** `Unable to find instance` (an object read when none are left: check `instance_exists()`), a variable
read before it was set (set it in Create, or read struct fields with `struct[$ "name"]`), or an index out of range.
An unhandled exception stops a GameMaker game.

**In every engine:** make the path fail safe, and apply game state (score, pickups, purchases) *before* anything
that can fail, so an error can't eat what the player earned.

## 4. Verify in the next version

Bump the version the game reports, ship the fix, and compare the error's sessions in the new version against the
old one once players have played it. A group that keeps appearing in the new version isn't fixed.

## With Glitchless

[Glitchless](https://www.glitchless.gg) (open source, MIT) does steps 1 and 4 for every engine above: errors
grouped by fingerprint and ranked by sessions, with breadcrumbs, device, version, source-mapped web stacks, and
reports players send from inside the game with a screenshot. A coding agent reads them over MCP
(`glitchless_triage`, `glitchless_get_error`, `glitchless_bug_reports`) and checks the fix with
`glitchless_compare_versions`. The `glitchless-setup` skill adds it to a game.
