# Chivalry 2 Mod Wiki

Reusable guidance for people and agents developing Chivalry 2 content with ArgonSDK and Unchained. This folder is standalone: copy it as a unit. It requires no personal workspace, private scripts, custom editor helpers, or example-mod assets.

These notes distill development observations recorded through **2026-11-11** (pages added or refreshed 2026-11-11: preflight checks, editor automation, cook and package, FFA map setup, Team Objective maps, mod audio, spawns step 3, CMW conversion; other pages date from 2026-10-20), primarily with **Unreal Engine 4.25.4**. They are a documentation snapshot, not a fresh compatibility audit. Version-specific observations must be checked against the reader's engine, SDK, launcher and game. Source experiments are summarized; their private logs and binaries are not distributed. A reported observation is not a reproducible test supplied by this wiki.

## Start here

Agents and quick lookups: the [agent index](AGENT-INDEX.md) routes a task to one page. Each folder has a README listing its pages.

### [Getting started](getting-started/README.md)

First steps: a Blueprint mod and an FFA map, from setup to a loaded PAK.

| Page | Contents |
| --- | --- |
| [First mod](getting-started/create-first-mod.md) | Blueprint actor, marker, loading and iteration |
| [First map](getting-started/create-first-map.md) | FFA arena, initial spawning and navigation |

### [Game modes](game-modes/README.md)

How matches, spawns and objectives work, and how a map sets them up.

| Page | Contents |
| --- | --- |
| [FFA map setup](game-modes/ffa-map-setup.md) | Official-style FFA: world settings, message table, kill cap, start timing, bot roaming, cinematics, map limits |
| [Spawns and navigation](game-modes/spawns-and-navigation.md) | Respawn waves, match startup and intended routes |
| [FFA lifecycle](game-modes/ffa-lifecycle.md) | Match state, spawning, possession/control, death, respawn and completion; interface evidence and gaps |
| [Team Objective maps](game-modes/team-objective-maps.md) | Staged TO match from a mod: stage chain, victory stages, per-stage spawns, objectives verified in game, untested parts |

### [Editor and scripting](editor-and-scripting/README.md)

Automating the UE 4.25.4 editor, authoring Blueprints from scripts, and cheap checks before expensive runs.

| Page | Contents |
| --- | --- |
| [Editor automation](editor-and-scripting/editor-automation.md) | Launch modes, hidden launches, resident editor, Python limitations and saved-state checks |
| [Blueprint authoring](editor-and-scripting/blueprint-authoring.md) | Graph semantics, SDK stubs and shipping behavior |
| [Preflight checks](editor-and-scripting/preflight-checks.md) | Cheap-first check ladder before editor runs and cooks; UE 4.25 behaviours that broke runs, with workarounds |

### [Visuals and audio](visuals-and-audio/README.md)

Lighting, materials, reflections and sound in the shipping game.

| Page | Contents |
| --- | --- |
| [Lighting and materials](visuals-and-audio/lighting-and-materials.md) | Static-lighting usage flags, baking and material pitfalls |
| [Mirrors and captures](visuals-and-audio/mirrors-and-captures.md) | Reflection techniques, performance and telemetry |
| [Mod audio](visuals-and-audio/mod-audio.md) | Mod sounds and the volume sliders: Wwise bank rule, packing, licence caveat, SoundMix fallbacks |

### [Packaging and hosting](packaging-and-hosting/README.md)

Cooking, packing, installing and publishing a mod; running a server.

| Page | Contents |
| --- | --- |
| [Cook and package](packaging-and-hosting/cook-and-package.md) | Fresh output, dependency closure, archive checks, loose files, library/map split, publishing format and installation |
| [Server hosting](packaging-and-hosting/server-hosting.md) | Setup sequence and version-specific travel pitfalls |

### [Game research](game-research/README.md)

Reading game data and reviewing PAKs without changing the game.

| Page | Contents |
| --- | --- |
| [Game data research](game-research/game-data-research.md) | Inheritance, ability tables and animation evidence |
| [PAK review](game-research/pak-review.md) | Static findings versus actual runtime behavior |

### [CMW conversion](cmw-conversion/README.md)

Staged UE3 → UE4 conversion of Chivalry: Medieval Warfare maps: source collection, geometry and collision, materials and environment, light maps, completion rules, packaging and verification.

### Repository files

- [Agent index](AGENT-INDEX.md): task → page routing.
- [Contributing](CONTRIBUTING.md): public-content boundary, evidence requirements, folder layout.
- [License](LICENSE): CC BY 4.0.

## Public starting points

- [Generated TBL API and Blueprint-node reference](https://github.com/Chiv2-Community/chiv2-tblcpp-obsidian-clean): Markdown class/type references, Blueprint pin tables and node images. Open the repository folder as an Obsidian vault for visual browsing, or read/search the files directly. Enable **Existing files only** in graph settings. Sampled API pages date to July 2023; compare installed SDK signatures and runtime evidence.

- [ArgonSDK](https://github.com/Chiv2-Community/ArgonSDK) and its [asset workflow wiki](https://github.com/Chiv2-Community/ArgonSDK/wiki).
- [ArgonSDK Field Guide](https://knutschbert.github.io/ArgonSDK-FieldGuide/).
- [Unchained documentation](https://unchained.wiki/) and [launcher releases](https://github.com/Chiv2-Community/UnchainedLauncher/releases).

Links identify upstream resources; their current availability and contents were not rechecked for this extraction. Prefer source and release notes matching your installed version when older tutorials disagree.

## License

Text © SageFlare, licensed under [Creative Commons Attribution 4.0 International](LICENSE) (CC BY 4.0): copy, adapt and share with credit. Chivalry 2, Chivalry: Medieval Warfare, Unreal Engine and the linked tools belong to their respective owners; this wiki contains no game assets.

Paths such as `/Game/Mods/AgMods/ExampleMod` are illustrative Unreal package paths. `<ProjectRoot>`, `<EngineRoot>`, `<OutputRoot>` and `<GameRoot>` mean locations chosen by the reader. They are not commands to paste unchanged.
