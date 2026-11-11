# Chivalry 2 modding notes

Reusable guidance for people and agents developing Chivalry 2 content with ArgonSDK and Unchained. This folder is standalone: copy it as a unit. It requires no personal workspace, private scripts, custom editor helpers, or example-mod assets.

These notes distill development observations recorded through **2026-10-20**, primarily with **Unreal Engine 4.25.4**. They are a documentation snapshot, not a fresh compatibility audit. Version-specific observations must be checked against the reader's engine, SDK, launcher and game. Source experiments are summarized; their private logs and binaries are not distributed. A reported observation is not a reproducible test supplied by this wiki.

## Start here

| Guide | Contents |
| --- | --- |
| [First mod](create-first-mod.md) | Blueprint actor, marker, loading and iteration |
| [First map](create-first-map.md) | FFA arena, initial spawning and navigation |
| [Editor automation](editor-automation.md) | Launch modes, Python limitations and saved-state checks |
| [Blueprint authoring](blueprint-authoring.md) | Graph semantics, SDK stubs and shipping behavior |
| [Lighting and materials](lighting-and-materials.md) | Static-lighting usage flags, baking and material pitfalls |
| [Spawns and navigation](spawns-and-navigation.md) | Respawn waves, match startup and intended routes |
| [FFA lifecycle](ffa-lifecycle.md) | Match state, spawning, possession/control, death, respawn and completion; interface evidence and gaps |
| [Cook and package](cook-and-package.md) | Fresh output, dependency closure, archive checks and installation |
| [Mirrors and captures](mirrors-and-captures.md) | Reflection techniques, performance and telemetry |
| [Server hosting](server-hosting.md) | Setup sequence and version-specific travel pitfalls |
| [Game data research](game-data-research.md) | Inheritance, ability tables and animation evidence |
| [PAK review](pak-review.md) | Static findings versus actual runtime behavior |
| [CMW conversion](cmw-imports/README.md) | Source collection, fidelity, collision and environment conversion |
| [Contributing](CONTRIBUTING.md) | Public-content boundary and evidence requirements |

## Public starting points

- [Generated TBL API and Blueprint-node reference](https://github.com/Chiv2-Community/chiv2-tblcpp-obsidian-clean): Markdown class/type references, Blueprint pin tables and node images. Open the repository folder as an Obsidian vault for visual browsing, or read/search the files directly. Enable **Existing files only** in graph settings. Sampled API pages date to July 2023; compare installed SDK signatures and runtime evidence.

- [ArgonSDK](https://github.com/Chiv2-Community/ArgonSDK) and its [asset workflow wiki](https://github.com/Chiv2-Community/ArgonSDK/wiki).
- [ArgonSDK Field Guide](https://knutschbert.github.io/ArgonSDK-FieldGuide/).
- [Unchained documentation](https://unchained.wiki/) and [launcher releases](https://github.com/Chiv2-Community/UnchainedLauncher/releases).

Links identify upstream resources; their current availability and contents were not rechecked for this extraction. Prefer source and release notes matching your installed version when older tutorials disagree.

Paths such as `/Game/Mods/AgMods/ExampleMod` are illustrative Unreal package paths. `<ProjectRoot>`, `<EngineRoot>`, `<OutputRoot>` and `<GameRoot>` mean locations chosen by the reader. They are not commands to paste unchanged.
