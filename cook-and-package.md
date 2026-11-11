# Cook, verify and install a mod

Use the Unreal tools matching your engine. Examples describe paths; supply your own filesystem roots.

| Item | Conventional location |
| --- | --- |
| Editable content | `<ProjectRoot>/Content/Mods/AgMods/ExampleMod` |
| Unreal package root | `/Game/Mods/AgMods/ExampleMod` |
| Cooked content | `<ProjectRoot>/Saved/Cooked/WindowsNoEditor/TBL/Content/Mods/AgMods/ExampleMod` |
| PAK mount root | `../../../TBL/Content/Mods/AgMods/ExampleMod/` |

If using a junction, cook through the mounted project package path. Verify its target rather than maintaining two editable copies.

## Build gates

1. Save authored content; bake when needed. Cold-compile Blueprints and validate saved assets in a fresh process.
2. Preserve or isolate the previous owned cook output so it cannot pass as a fresh build. Avoid overlapping editor/cook jobs.
3. Cook explicit maps using `-run=cook -targetplatform=WindowsNoEditor -ForceDependsGathering -map=<PackagePath1>+<PackagePath2>`. For actor-only mods, ensure their asset directory is selected by the project's cook configuration.
4. Require a successful exit and inspect the log, including the `Success - 0 error(s)` result. Investigate owned-content warnings. Reconstructed game-class references can produce substantial unrelated warning noise.
5. Require expected fresh maps, Blueprints, enums, BuiltData and generated `.uexp`/`.ubulk` companions. Never substitute uncooked editor assets.
6. Create a response file mapping each owned cooked file to its intended mount path. Run `UnrealPak <pak> -Create=<response-file> -Compress`, then `-Test` and `-List`.
7. Compare actual archive paths and counts with the intended file inventory. Record size, SHA256, dependencies, tool versions and validation status.

Shared cook roots can include stale experimental assets unrelated to the selected map. Diagnose those references before changing content; temporarily exclude only known owned test content and restore it reliably.

Some inspected SDK packaging wrappers failed to propagate tool errors, and one cook wrapper could invoke cooking twice. Inspect your version's implementation; require real exit codes, fresh output and archive checks rather than trusting a wrapper's printed file count.

## Dependencies

Scan package dependencies, including maps through AssetRegistry. Include required custom assets or document their separately installed dependency PAKs. Do not bundle reconstructed copies of original game assets; retain appropriate never-cook exclusions.

An archive hash proves identity, not completeness. An editor-only importer/helper reference leaking into runtime packages is a release failure. Remove unnecessary import metadata after checking its role and revalidate dependency closure.

## Installation

Install only into a confirmed game installation selected for testing. Wait for the game and server to close; do not terminate another user's session automatically. Back up the specific prior mod PAK, copy the verified replacement, and compare installed/build SHA256. Leave unrelated PAKs alone.

Record the installed version and rollback artifact privately. Test loading and intended behavior in the shipping game; archive integrity cannot establish runtime compatibility.
