# Stages, contracts and release

## Stage separation

- Each stage reads the previous stage's output and never changes it. A reader can stop after any stage: a collection or a faithful base is usable on its own.
- Stages 1 and 2 do not package. A package from stage 2 or 3 is a test build behind an explicit opt-in; release packaging is its own step.
- Stage 2 is faithful to **what the final map uses**, not to CMW engine-era processing. Distance culling (cull-distance volumes, foliage instance cull, emitter draw distances) was dropped and recorded; stage 1 still collects it. Placement-bearing gameplay actors (starts, objectives) stay as records.
- Anything without a direct conversion goes to stage 3 as a named item: a flag, never a silent substitute.

## One method per feature

Use one conversion method per feature for every map. Where maps need different handling, branch on a **source flag or property** recorded with evidence, never on the map's name. Examples that held across maps:

- `LightMapCoordinateIndex` missing from a mesh export: the light map is in UV0 (the exporter omits default values).
- `TwoSidedLightingMask` present: the material needs a transmission term.
- Landscape normal input unconnected: use the heightmap normal only.

Fix a stage failure as a general rule and re-run, rather than patching one map.

## Bundle requirements (stage 1)

- Key objects by package/group/object identity, never short names or exported filenames. Several CMW packages define materials with the same short name; resolving by name bound the wrong one.
- Record schema, tool versions, input identities, outputs, coverage and completion state. References inside the bundle are relative to its root.
- Keep original bytes and typed evidence, including unsupported payloads.
- Include streamed levels (follow them recursively), component-local transforms, class defaults, material dependencies, collision policies and environment state links. Find visual mesh components on every owner class, including gameplay actors.
- A missing input becomes a typed request to the collector and a new bundle revision. A later stage must not quietly read the installed game.
- Size a map by all its levels, including art sub-levels, not the persistent package.

## Starting state

- **Unsaved values are class defaults.** Read the CMW SDK UnrealScript `defaultproperties`. An unset bool is false, not true. Examples: `InterpActor` defaults `bCollideActors` to false; `InstancedFoliageActor` sets no collision.
- **Omitted struct members** take `structdefaultproperties` (for example LinearColor alpha 1).
- **Gates and barriers** keep their authored state: closed stays closed. Opening them is manual or a stage 3/4 rule.
- **Streaming levels:** place what the source has loaded once play settles with players present but idle. Evaluate the Kismet graph with no input: fire level-loaded events, one player spawn, sequence activations and remote events; evaluate delays, gates, integer compares and level streaming; stop at every branch that needs input. Place the always-loaded levels plus the levels loaded at the end of that run.
  - Placing every alternative level stacks them. Placing none can leave holes, because alternatives can hold structure the always-loaded art needs.
  - Actors that Kismet hides, moves or re-collides after map start keep their map-start state and are recorded.

Evidence: class defaults and gate states editor and in game; the settled streaming-level rule editor only (not separately checked in game on a map with alternative levels).

## Packaging and release

**Library and map split.** Assets used by several maps go to one shared library; each map package holds only what is map-specific.

- The library never references a map. Check this on the full dependency closure.
- Anything that carries a map's own data stays map-local: light-map adapter materials, foliage meshes and their materials.
- Reuse a library asset only when it is provably the same: equal material specification, equal mesh material bindings and slot layout, equal texture settings. Otherwise import a map-local copy or a recorded variant.
- To repoint references, copy the asset, add session-only core redirects (package and object), re-save, then end the editor session (UE 4.25 cannot remove a redirect list).
- Do not consolidate assets for this. In UE 4.25 `consolidate_assets` re-points hard references only; soft references (for example material texture streaming data) still need the redirector. The cooker does not cook redirector-only packages and rewrites cooked soft paths to the destination. `find_object` follows redirectors unless told not to.

**Prune unused content.** The map is the package's only entry point. Keep only mod packages reached through cooked name tables, including FName numbered siblings (`<name>_<n>`), and list the rest. No manual inspection was needed.

**Chunks and installation.**

- Split large releases into chunk archives below the hosting size limit. We used at most 1.8 GB per chunk against a 2 GB limit. Keep a stable unit-to-chunk assignment, and round-trip each chunk (extract, compare bytes).
- Install the library before the maps that depend on it.
- UE4 mounts the `Paks` folder recursively, so library chunks may sit in a subfolder. With the launcher version we used, Unchained listed mods from the `Paks` root only, so put map packages in the root. Check this against your launcher version.
- One merged library archive and many per-package chunks gave the same game-start and map-load times (in game, measured from the Unchained log).

General cook and archive checks: [cook and package](../cook-and-package.md), [PAK review](../pak-review.md).
