# Agent index

[Home](README.md) · [Contributing](CONTRIBUTING.md)

For AI agents (and people) working on Chivalry 2 mods with ArgonSDK (UE 4.25.4) and Unchained. Maintained with the wiki: every new page gets a row here.

## How to use this wiki

1. Do not preload the whole wiki. Find the task below, read its **Start** page, and only the sections you need. Unsure which task fits: read one section README from the table below, then one page.
2. Open the **Related** page only when the task touches it. Search exact symbols before tracing anything again, e.g. `rg -n -i "SpawnSequence|GroupSpawner" .`
3. Keep evidence levels apart: documentation/source inspection, editor (saved state reopened), package (cooked archive), in game. Pages state which level supports each claim; an editor or package result does not prove gameplay.
4. SDK classes are partly reconstructed stubs: never infer shipping-game behaviour from SDK function bodies or return values. Test in game.
5. These pages describe methods, not a supplied toolchain. Paths like `/Game/Mods/AgMods/ExampleMod` and `<ProjectRoot>` are placeholders.

## Sections

Repository root holds only this index, the [README](README.md), [CONTRIBUTING](CONTRIBUTING.md) and [LICENSE](LICENSE). Every page lives in one topic folder:

| Folder | Covers |
| --- | --- |
| [getting-started/](getting-started/README.md) | first Blueprint mod, first FFA map |
| [game-modes/](game-modes/README.md) | FFA setup and lifecycle, spawns and navigation, Team Objective maps |
| [editor-and-scripting/](editor-and-scripting/README.md) | editor automation, Blueprint authoring from scripts, preflight checks and UE 4.25 limits |
| [visuals-and-audio/](visuals-and-audio/README.md) | lighting and materials, mirrors and captures, mod audio |
| [packaging-and-hosting/](packaging-and-hosting/README.md) | cook, package, install, publish; server hosting |
| [game-research/](game-research/README.md) | read-only game data research, PAK review |
| [cmw-conversion/](cmw-conversion/README.md) | converting Chivalry: Medieval Warfare (UE3) maps |

## Route a task

| Task | Start | Related | Keywords |
| --- | --- | --- | --- |
| First Blueprint mod | [First mod](getting-started/create-first-mod.md) | [Cook and package](packaging-and-hosting/cook-and-package.md) | `ArgonSDKModBase`, `DA_ModMarker`, junction, iterate |
| First FFA map | [First map](getting-started/create-first-map.md) | [Spawns and navigation](game-modes/spawns-and-navigation.md) | `Mode_FFA`, `gamemode_type=FREE_FOR_ALL`, `BaseSpawner`, `NavMeshBoundsVolume` |
| Official-style FFA match | [FFA map setup](game-modes/ffa-map-setup.md) | [FFA lifecycle](game-modes/ffa-lifecycle.md) | world settings, kill cap, `ScoredStage`, start timing, bot roaming, cinematics |
| Spawns, respawn waves, bots | [Spawns and navigation](game-modes/spawns-and-navigation.md) | [FFA map setup](game-modes/ffa-map-setup.md) | `GroupSpawner`, `SpawnSequence`, waves, startup, navigation |
| How an FFA match runs (game state, possession, death) | [FFA lifecycle](game-modes/ffa-lifecycle.md) | [Spawns and navigation](game-modes/spawns-and-navigation.md) | readiness, queuers, possession, respawn, match end |
| Team Objective map | [Team Objective maps](game-modes/team-objective-maps.md) | [Spawns and navigation](game-modes/spawns-and-navigation.md) | `StagedGameMode`, stage chain, victory stage, per-stage spawns, capture area, escort |
| Script the editor (Python, commandlets) | [Editor automation](editor-and-scripting/editor-automation.md) | [Preflight checks](editor-and-scripting/preflight-checks.md) | headless, hidden launch, resident editor, Python 2.7 API gaps, saved state |
| Avoid wasted editor runs / UE 4.25 limits | [Preflight checks](editor-and-scripting/preflight-checks.md) | [Editor automation](editor-and-scripting/editor-automation.md) | lint, compile, canary, review, engine behaviour table |
| Blueprints from scripts or SDK stubs | [Blueprint authoring](editor-and-scripting/blueprint-authoring.md) | [Editor automation](editor-and-scripting/editor-automation.md) | graph rules, construction script, cold compile, user enums, nativization |
| Lighting and materials | [Lighting and materials](visuals-and-audio/lighting-and-materials.md) | [Preflight checks](editor-and-scripting/preflight-checks.md) | `bUsedWithStaticLighting`, Lightmass, emissive, material quirks |
| Mirrors and scene captures | [Mirrors and captures](visuals-and-audio/mirrors-and-captures.md) | [Lighting and materials](visuals-and-audio/lighting-and-materials.md) | scene captures, planar reflections, owner visibility, cost, telemetry |
| Mod sounds and the volume sliders | [Mod audio](visuals-and-audio/mod-audio.md) | [Cook and package](packaging-and-hosting/cook-and-package.md) | Wwise bank, buses, loose files, `SoundMix` fallback |
| Cook, package, install, publish | [Cook and package](packaging-and-hosting/cook-and-package.md) | [PAK review](game-research/pak-review.md) | cook scope, dependencies, loose files, library PAKs, install, registry `mod.json` v1 |
| Review a PAK (content vs runtime) | [PAK review](game-research/pak-review.md) | [Cook and package](packaging-and-hosting/cook-and-package.md) | archive inventory, asset shadowing, runtime evidence |
| Read game data (no game changes) | [Game data research](game-research/game-data-research.md) | [Blueprint authoring](editor-and-scripting/blueprint-authoring.md) | inheritance, ability tables, animation timing |
| Host a server / travel problems | [Server hosting](packaging-and-hosting/server-hosting.md) | | baseline, version-specific travel |
| Convert a Chivalry: Medieval Warfare (UE3) map | [CMW conversion](cmw-conversion/README.md) | [Passes and release](cmw-conversion/workflow.md) | collection, base conversion, completion, objective setup, release |
| CMW meshes, BSP, terrain, collision | [Geometry and collision](cmw-conversion/geometry-and-collision.md) | [Tools and evidence](cmw-conversion/tools-and-evidence.md) | LODs, compiled BSP, terrain, hulls, volumes |
| CMW materials, decals, fog, effects, audio | [Materials and environment](cmw-conversion/materials-and-environment.md) | [Completion rules](cmw-conversion/completion-rules.md) | material graphs, decals, post-process, emitters, cues |
| CMW precomputed lighting | [Light maps](cmw-conversion/light-maps.md) | [Materials and environment](cmw-conversion/materials-and-environment.md) | LightMap2D, ShadowMap2D, build data, atlas, Lightmass |
| Make a converted map playable (FFA) | [Completion rules](cmw-conversion/completion-rules.md) | [FFA map setup](game-modes/ffa-map-setup.md) | atmosphere, scale, Wwise, replacements, traps, FFA starts |
| CMW tools, verification, crash triage | [Tools and evidence](cmw-conversion/tools-and-evidence.md) | [Preflight checks](editor-and-scripting/preflight-checks.md) | UModel, UELib, verification levels, failure handling |
| Add or change a page | [Contributing](CONTRIBUTING.md) | this index | content boundary, evidence, review before sharing |

## External references

API/class lookups: the generated TBL reference and ArgonSDK repositories linked from the [README](README.md#public-starting-points). Compare their signatures with the installed SDK; they are interface evidence, not shipping behaviour.
