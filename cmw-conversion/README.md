# Converting Chivalry: Medieval Warfare content

[Home](../README.md) · [Agent index](../AGENT-INDEX.md)

Methods and engine facts for converting Chivalry: Medieval Warfare (CMW, Unreal Engine 3) maps into Chivalry 2 mods built with ArgonSDK (UE 4.25.4) and loaded through Unchained. The methods below were used to convert several official CMW maps, duel and Team Objective layouts, into playable Chivalry 2 FFA maps. **The tooling that did this is private and is not supplied here.** The pages describe algorithms, rules and checks, so you can reproduce them with your own tools.

A `.udk` map cannot be opened in UE4. Conversion means extracting assets and rebuilding the scene. Start from a legally available CMW installation: keep the full cooked package set and its texture caches, not just the map package. Treat the installation and every extraction as read-only.

Observed source serialization on PC: package version 860, licensee 0, engine 10246. That tuple alone does not certify that a parser supports compression or every native payload.

## Conversion passes (a suggested split)

Splitting the work into passes that each read the previous output and never change it kept fixes cheap to re-run and made each result usable on its own. These pages name the passes as below; the split is a recommendation, not a requirement.

| Pass | Output | Boundary |
| --- | --- | --- |
| Collection | Original resources, typed scene data, class defaults, dependencies, unresolved raw evidence | No conversion decisions |
| Base conversion | UE4 map that renders, collides and sounds like the source at source scale; objectives inactive, authored starting states kept | No approximations and no Chivalry 2 replacements; anything without a direct conversion becomes a record |
| Completion | Playable FFA map produced by ordered, automatic, rule-based steps | Same rules on every map; no per-map tuning |
| Objective setup (Team Objective) | Objectives, stages and team spawns from CMW objective data | **Proposal, not built** |
| Release | Shared library package set plus one package set per map | Unused content pruned; checked before installation |

Manual refinement (exposure, extra spawns, opening barriers, cameras) happens after completion, never inside an automatic pass. See [passes and contracts](workflow.md).

## Pages

| Page | Scope |
| --- | --- |
| [Passes, contracts and release](workflow.md) | Pass separation, source-flag branching, starting state, streaming levels, library/map packaging |
| [Geometry and collision](geometry-and-collision.md) | Meshes, LODs, compiled BSP, terrain, skeletal props, hulls, volumes, helper geometry |
| [Materials and environment](materials-and-environment.md) | Material graphs, usage flags, decals, post-process, fog, effects, audio |
| [Light maps](light-maps.md) | CMW light, shadow and vertex light maps: adapter materials, import as UE4 build data, Lightmass |
| [Completion rules](completion-rules.md) | Completion rules (atmosphere, scale, Wwise sound, replacements, map logic, FFA) and the objective-setup proposal |
| [Tools and evidence](tools-and-evidence.md) | Public tools and their limits, verification levels, failure handling, crash triage |

## Evidence levels used here

- **source**: CMW SDK UnrealScript/shader source, UE 4.25 engine source or package inspection.
- **editor**: saved assets reopened and checked in a separate editor process.
- **package**: cooked archive checks (content, dependencies, integrity).
- **in game**: a player loaded the installed map in Chivalry 2 with Unchained and checked what is named. "Looks good" covers only what that person looked at.

Successful saving, matching hashes, actor counts or an empty work queue never establish visual fidelity or gameplay.

## Not covered

- No automatic conversion of CMW objective logic (objective setup is a proposal).
- No supplied parser, editor plugin or command. Several steps need native editor code because UE 4.25 Python does not expose the required API; the pages say where.
- Exact visual parity is not claimed. Each page lists what stays approximate or unverified.

This edition replaces the 2026-10-20 pilot notes. Those described a staged design as a proposal and baked lighting only through Unlit display materials. Both are now superseded: collection, base conversion and completion were used on complete maps (objective setup is still a proposal), and CMW light maps can be imported as native UE4 build data.
