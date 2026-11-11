# Blueprint authoring and validation

Use the Blueprint editor for graph authoring. UE 4.25's standard Python interface does not expose general K2 graph creation. This guide assumes no custom graph plugin.

## Graph behavior

The public [TBL API and Blueprint-node reference](https://github.com/Chiv2-Community/chiv2-tblcpp-obsidian-clean) supplies generated signatures and node pin tables/images. Class pages live in `docs/SDK/Source/Classes/`; Blueprint nodes live in `TBL_src_clean/<Class>/nodes/`. Read directly or open as an Obsidian vault. Historical interface documentation does not establish current shipping behavior.

- Require a clean compile and investigate pruned nodes. Impure calls need execution wiring; pure calls have no execution pins. Check actual node signatures in the installed engine.
- Pure expressions can be reevaluated for each consumer. Cache expensive repeated results once per intended update, while preserving evaluation timing.
- Resolve wildcard types through connections and inspect the resulting pins. Do not infer types from node display names.
- When changing function signatures, refresh or rebuild all callers. Stale self-call nodes can survive an in-session compile.
- Avoid deleting parent assets with live child or typed references. Back up owned assets and prefer controlled in-place edits.
- After restructuring, check real ticks and observable results, not only graph appearance.

## Saved instances

Changing a placed Blueprint component through `set_editor_property` can rerun its construction script. References may become stale and edits may be overwritten. Reacquire references and validate after a cold reload. Runtime-only state often belongs in BeginPlay.

Validate saved instances without spawning and mutating construction-script actors in headless validators; that pattern caused crashes in the tested setup. A fresh-process compile and saved-state inspection are stronger evidence than an in-session up-to-date status.

## SDK versus game

Reconstructed game Blueprints can be stubs with empty functions or missing components. A matching SDK class name does not establish game behavior. Reference the correct shipped class path and test in the shipping game.

Inherited component overrides can depend on `VariableGuid`; a reconstructed parent GUID that differs from the game's can cause a child override to be ignored. Placed-instance overrides behaved differently and need their own verification.

`Print String` and editor Kismet-event console calls are not dependable shipping features. Use a visible gameplay checkpoint, `Show Local Chat` where available, or deliberate SaveGame telemetry.

## User enums and structs

Nativization can omit user-defined enums and structs needed by a mod. With the ArgonSDK project setting `BlueprintNativizationMethod=Exclusive`, a cook converts every user-defined enum and struct to C++ unless it is excluded: the "only selected assets" rule applies to Blueprints only (UE 4.25 `BlueprintNativeCodeGenModule.cpp`). The converted asset is missing from the cooked content while your Blueprints still import it, and the cook still reports success.

Fix: list each asset under `[BlueprintNativizationSettings]` as `+ExcludedAssets=/Game/Mods/<YourMod>/<Path>/<AssetName>` in `DefaultEditor.ini`, then recook. Use one line per asset; `ExcludedFolderPaths` only applies to Blueprint classes. Then require the type's cooked `.uasset`/`.uexp` in the actual PAK listing.

Evidence: enums at package level (found when a mod's archive lacked them; the exclusion restored them); structs at package level in a dedicated A/B cook (UE 4.25.4): an excluded struct and enum were cooked and packed, while non-excluded controls were converted to C++ and absent from the PAK although the holder Blueprint still imported them. The struct case was not loaded in game. Source caveat: an excluded asset can still be converted if it has a circular reference with a converted asset.

Save enum assets explicitly. Python name normalization and default ordering can differ from display labels; verify the saved value rather than assigning an arbitrary integer.
