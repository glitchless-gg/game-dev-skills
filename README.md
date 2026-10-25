# Game dev skills for coding agents

[![skills.sh installs](https://skills.sh/b/glitchless-gg/game-dev-skills)](https://www.skills.sh/glitchless-gg/game-dev-skills)

Skills that teach Claude Code, Codex, Cursor and other coding agents how to find and fix what players actually
hit: stutters, crashes, slow starts, leaks, bad reviews and where players quit. Each one measures first, changes
one thing, and proves the fix with the same measurement.

They work on their own with each engine's own profiler. Where real-player numbers help, they show how to get them
with [Glitchless](https://www.glitchless.gg), the open-source game observability tool these come from.

## Install

```bash
npx skills add glitchless-gg/game-dev-skills
```

Or one skill:

```bash
npx skills add glitchless-gg/game-dev-skills --skill godot-performance
```

For Claude Code, the [Glitchless plugin](https://github.com/glitchless-gg/glitchless) bundles the Glitchless
skills with its MCP server.

## Skills

### Performance

| Skill | For |
| --- | --- |
| [game-performance](skills/game-performance/SKILL.md) | Any engine: the frame budget, p95 vs average fps, hitches vs slow frames, the measure-fix-measure loop |
| [threejs-performance](skills/threejs-performance/SKILL.md) | Three.js and React Three Fiber: `renderer.info`, instancing, `dispose()`, `compileAsync`, shadows, pixel ratio |
| [godot-performance](skills/godot-performance/SKILL.md) | Godot 4: Profiler and Monitors, pipeline compiles, orphan nodes, threaded loading, the Compatibility renderer on Android |
| [unity-performance](skills/unity-performance/SKILL.md) | Unity: profiling a development build, GC allocations, pooling, shader variants, async scene loads |
| [unreal-performance](skills/unreal-performance/SKILL.md) | Unreal Engine 5: PSO hitches, garbage collection, level streaming, Blueprint Tick, the Steam Deck |
| [phaser-performance](skills/phaser-performance/SKILL.md) | Phaser 3 and 4: Groups instead of new objects, listeners that pile up on restart, textures, frozen game loops |

### Crashes, reviews and stores

| Skill | For |
| --- | --- |
| [game-crash-triage](skills/game-crash-triage/SKILL.md) | Errors and crashes from players, grouped and ranked, with each engine's usual culprits |
| [steam-review-triage](skills/steam-review-triage/SKILL.md) | Which build turned Steam reviews negative, Steam Deck players vs desktops, beta branches |
| [mobile-store-vitals](skills/mobile-store-vitals/SKILL.md) | Android vitals (slow rendering, ANRs, crashes) and Apple's hangs and launch time, by version |
| [steam-deck-testing](skills/steam-deck-testing/SKILL.md) | Send a build to a Steam Deck over the network, start it, read its logs and measure it there |

### Analytics

| Skill | For |
| --- | --- |
| [game-analytics](skills/game-analytics/SKILL.md) | Turn the developer's questions into events, funnels and a dashboard, without tracking players |

### Glitchless

| Skill | For |
| --- | --- |
| [glitchless-setup](skills/glitchless-setup/SKILL.md) | Add Glitchless to a Three.js, Phaser, Babylon.js, PlayCanvas, Unity, Godot, Unreal or GameMaker game |
| [glitchless-triage](skills/glitchless-triage/SKILL.md) | Find the most important problem from real players' data, fix it, verify with a playtest |
| [glitchless-dashboard](skills/glitchless-dashboard/SKILL.md) | Design the game's own dashboard: the collectors, then the pages and panels |

## Contributing

Issues and pull requests are welcome. A good skill change comes with what it fixed: the engine, the symptom, the
numbers before and after. The four Glitchless skills (`glitchless-*` and `steam-deck-testing`) are copied from the
plugin in [glitchless-gg/glitchless](https://github.com/glitchless-gg/glitchless), so change them there.

MIT licensed.
