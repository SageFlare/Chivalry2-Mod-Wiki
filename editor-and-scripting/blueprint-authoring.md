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

Rules found while generating and testing graphs in UE 4.25.4 (editor compile; the runtime ones were seen in the shipping game):

- Const `BlueprintCallable` functions (for example `GetViewportSize`, `DeprojectScreenPositionToWorld`) are pure: do not wire exec. `GetCurrentLevelName`, `GetActorOfClass` and `LoadClassAsset_Blocking` are impure in 4.25: unwired, they are pruned and the Blueprint still compiles with warnings.
- `ForEachLoop` pins are `Exec`, `Array`, `LoopBody`, `Array Element`, `Array Index`, `Completed`; `ReverseForEachLoop` names them `ArrayElement` / `ArrayIndex` (no space). The generic `Array_Length` node could leave an unresolved wildcard; tracking the last index avoided it.
- A class output (`GetObjectClass`) does not connect to an object input (`GetDisplayName`); use `GetClassDisplayName`.
- Latent nodes (`Delay`, `RetriggerableDelay`) work only in event graphs. A loop that calls something able to restart the same handler (for example `AddScore` completing a stage, which starts the next stage's handlers) should not act inside its body: find the row in the loop and act once after `Completed`.
- Pure inputs are all evaluated. A `BooleanAND` does not short-circuit, so a native call fed a possibly-None object (such as the local pawn on a dedicated server) must sit behind exec `IsValid` branches, not behind a pure AND.
- Timers by name (`K2_SetTimer` with `FunctionName`) compile even with a typo. Check each name against the Blueprint's functions.
- Never compare a per-tick distance or angle to a fixed threshold; divide by `DeltaSeconds`. A 0.5 uu-per-frame "moving" test failed at an uncapped 240+ FPS in the shipping game. On a client, replicated movement arrives only on network-update frames, so decide "moving" from the time since the last observed move.
- Extra function return nodes may carry only constant defaults; a linked output on a second return node broke on cold reload.
- Game classes that the SDK lacks can be referenced by soft class path, loaded in game with `LoadClassAsset_Blocking` (wire its exec), cast with a class cast, then spawned. Editor Python reads such a path back as None; check the saved map bytes for it.
- Exported graph text escapes apostrophes, so text-based checks on comment text should avoid them.

## Saved instances

A level-placed actor keeps its own saved values. In a tested case, a placed Blueprint actor still had `bReplicates=False` (and its old tick interval) after the class default was changed, so it never replicated to clients. Before a multiplayer test, check replication settings on each placed instance, not only on the Blueprint class. `bReplicates`, `bAlwaysRelevant` and `bReplicateMovement` are edit-defaults-only properties, so scripts cannot set them on instances through the normal property setter.


Changing a component of a placed Blueprint actor through `set_editor_property` reruns its construction script and replaces its components. A component reference held from before becomes a `TRASH_*` object (owner None, collision profile None), and every later edit through it is silently lost: in a tested map a collision profile and a zone offset never reached the saved actor (editor, 2026-11-15). Look the component up again for each edit, read the value back, and validate after a cold reload. Rehearsals on a Blueprint template cannot catch this; rehearse on a spawned or placed instance. Runtime-only state often belongs in BeginPlay. Overriding the inherited `StaticMesh` on placed game actors did persist.

Before making an instance-editable variable internal (Disable Edit On Instance), migrate the placed instances first: in a tested case their overridden values read as class defaults after the change. Clearing expose-on-spawn also removes the matching `SpawnActor` pin.

Validate saved instances without spawning and mutating construction-script actors in headless validators; that pattern caused crashes in the tested setup. A fresh-process compile and saved-state inspection are stronger evidence than an in-session up-to-date status.

## SDK versus game

Reconstructed game Blueprints can be stubs with empty functions or missing components (breakable windows and gates, spawn queuers and stages among them). The SDK's native game functions (`TBL/Private/*.cpp`) are empty stubs as well, so an editor probe cannot show what a native game function does. A matching SDK class name does not establish game behavior. Reference the correct shipped class path and test in the shipping game.

Inherited component overrides can depend on `VariableGuid`; a reconstructed parent GUID that differs from the game's can cause a child override to be ignored. Placed-instance overrides behaved differently and need their own verification.

Do not call the engine's `Get All Actors With Tag` from Gameplay Statics (`GameplayStatics::GetAllActorsWithTag`) in a Chivalry 2 mod Blueprint. In our tests it crashed the shipping game every time it ran (an access violation, the same address in every crash): a few seconds after a trigger when called during play, and in the loading screen when called on BeginPlay. A pad that ran only this node, with nothing else, crashed as well. Use the game's own `Get All Actors With Tag` from `TBLSystemLibrary`, which has the same pins and is the only tag lookup the shipped level scripts use. With it, grouped HUD markers (`MultiActorHUDMarker`) mark placed and runtime-spawned actors without trouble, through `AddMarkedActor` or the server call `AuthoritySetMarkedActors`. Finding actors by class (`Get All Actors Of Class`) was not affected.

Objective markers (verified in the shipping game 2026-11-12 to 11-14, including a dedicated server): give each objective actor its own `HudMarkerDisplayComponent` (start hidden, replicated, shown with a server-side `SetVisible`), as retail capture areas and pushables do, and fill its per-team display info on each placed instance. A grouped `MultiActorHUDMarker` (placed, or your own subclass) only chooses among actors that already carry such a component; it showed up to two icons, removed an icon when its actor broke and cleared at stage end. Cut clutter on the per-actor component with retail options such as `MaxVisibleDistance`, or `bUseItemVisibility` with `InventoryItemClass` and `VisibilityCondition` (shown only while the player holds an item). Marker icons are soft paths to game textures (`/Game/UI/...T_ObjIcon_*`) that the SDK lacks: editor Python reads them back as None and the cook only warns, so check the saved map for the path. In our tests, crashes that first looked tied to placed marked actors or marker subclasses all came from the engine tag lookup above; without it, both worked.

`Print String` and editor Kismet-event console calls are not dependable shipping features. Use a visible gameplay checkpoint, `Show Local Chat` where available, or deliberate SaveGame telemetry.

## User enums and structs

Nativization can omit user-defined enums and structs needed by a mod. With the ArgonSDK project setting `BlueprintNativizationMethod=Exclusive`, a cook converts every user-defined enum and struct to C++ unless it is excluded: the "only selected assets" rule applies to Blueprints only (UE 4.25 `BlueprintNativeCodeGenModule.cpp`). The converted asset is missing from the cooked content while your Blueprints still import it, and the cook still reports success.

Fix: list each asset under `[BlueprintNativizationSettings]` as `+ExcludedAssets=/Game/Mods/<YourMod>/<Path>/<AssetName>` in `DefaultEditor.ini`, then recook. Use one line per asset; `ExcludedFolderPaths` only applies to Blueprint classes. Then require the type's cooked `.uasset`/`.uexp` in the actual PAK listing.

Evidence: enums at package level (found when a mod's archive lacked them; the exclusion restored them); structs at package level and in game in a dedicated A/B test (UE 4.25.4): an excluded struct and enum were cooked and packed, while non-excluded controls were converted to C++ and absent from the PAK although the holder Blueprint still imported them. In game, a probe Blueprint wrote a value into the excluded struct at runtime, read it back through a variable of that type and branched on it correctly. Source caveat: an excluded asset can still be converted if it has a circular reference with a converted asset.

Tested uses of an excluded user struct in Chivalry 2 (2026-11-11, [test repository](https://github.com/SageFlare/StructNativizationTest)): runtime write and read, struct-asset and Blueprint default values, values set per placed actor, arrays of structs, data tables with the struct as row type, SaveGame (persists across map loads and game restarts) and live replication to a client that joined a server.

Newer ArgonSDK packaging tooling is reported (by its developers, 2026-11) to exclude a mod's enums and structs from nativization automatically. This was not verified here; whichever tool cooks, require the type's files in the PAK listing.

Re-creating a user struct breaks data saved with it. Members are stored as `Name_N_<GUID>`; deleting and re-creating the struct gives new GUIDs, so values saved in other maps no longer match. Keep the struct asset and evolve it in place, or re-save every map that stored values with it. Adding a member at the end keeps saved values, and old elements read the new member's default. Never change an existing member's default: elements equal to the old default were not saved and follow the new one (editor and package level, 2026-11-12).

Struct arrays on placed actors can be written from Python by text; object paths that do not resolve import as None while the call still reports success, so read values back. Text forms that worked: enum member `NewEnumeratorN`, text `INVTEXT("...")`, string without quotes (quotes are stored literally), actor by its path name.

Save user structs by path: `EditorAssetLibrary.save_asset(path, False)`. The generic `save_loaded_asset(obj, False)` refuses a `UserDefinedStruct` ("Failed to validate the source. The object is of the base class type") and returns False, which stops a re-save loop over mixed assets. Saving by path worked for structs, Blueprints, materials and meshes (editor, 2026-11-15). `UserDefinedStruct` is not a 4.25 Python class; `create_asset(name, path, None, StructureFactory())` creates one.

Save enum assets explicitly. Python name normalization and default ordering can differ from display labels; verify the saved value rather than assigning an arbitrary integer.
