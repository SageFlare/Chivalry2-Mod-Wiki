# Cook, verify and install a mod

Use the Unreal tools matching your engine. Examples describe paths; supply your own filesystem roots. Before a cook, climb the [preflight checks](../editor-and-scripting/preflight-checks.md).

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
4. Require a successful exit and inspect the log, including the `Success - 0 error(s)` result. Investigate owned-content warnings. Reconstructed game-class references can produce substantial unrelated warning noise (`imported Serialize:/Game/... never saved as an export`).
5. Require expected fresh maps, Blueprints, enums, BuiltData and generated `.uexp`/`.ubulk` companions. Never substitute uncooked editor assets.
6. Create a response file mapping each owned cooked file to its intended mount path. Run `UnrealPak <pak> -Create=<response-file> -Compress`, then `-Test` and `-List`.
7. Compare actual archive paths and counts with the intended file inventory. Record size, SHA256, dependencies, tool versions and validation status.

Some inspected SDK packaging wrappers failed to propagate tool errors, and one cook wrapper could invoke cooking twice. Inspect your version's implementation; require real exit codes, fresh output and archive checks rather than trusting a wrapper's printed file count.

## Every cook touches every mod

The inspected ArgonSDK `Config/DefaultGame.ini` contains `+DirectoriesToAlwaysCook=(Path="/Game/Mods")`. Every cook therefore rewrites the cooked folders of every mod linked into the project, including SDK examples:

- A broken asset in any linked mod can fail any cook, and its errors appear in your log. Scope log scans to your own package roots.
- Timestamps of other mods' cooked files say nothing about leaks. Scan your own cooked packages for `/Game/Mods/<other mod>/` references instead.
- Cooking selected maps also produced `_BuiltData` for unrelated maps. Package only your owned folder and fail on any unexpected file there.

Shared cook roots can also include stale experimental assets. Diagnose those references before changing content; temporarily exclude only known owned test content and restore it reliably.

## Loose files

Files that are not cooked assets can be added to the same response file. A Wwise sound bank, for example, is mapped to `../../../TBL/Content/WwiseAudio/Windows/<Bank>.bnk`; in a shipping-game test, Chivalry 2 loaded such a bank by name. Count loose files in the inventory check.

## Dependencies and library PAKs

Scan package dependencies, including maps through AssetRegistry. Include required custom assets or document their separately installed dependency PAKs. Do not bundle reconstructed copies of original game assets; retain appropriate never-cook exclusions.

When maps share a library PAK, keep structural geometry in each map's own PAK. In a game test, a map whose structural geometry used library meshes dropped the player into the void when the library was absent. Only optional features should depend on the library. Further rules from a split into map PAKs plus library PAKs (archive-verified; layout shown loading in game):

- The map PAK holds only the map mod; a library PAK may reference only the library.
- Assign each library package to exactly one PAK and keep that assignment stable; no path may appear in two PAKs. Ship the reference closure of what each map reaches.
- Install library PAKs before map PAKs. The engine mounts PAKs in subfolders of `Paks`, but the tested Unchained launcher looked only in the `Paks` root for mods, so keep map PAKs in the root.
- One merged library PAK and many per-folder library PAKs gave the same measured load times on one map over several game sessions (which session used which layout was not logged; the numbers matched either way), so the choice is about packaging.

An archive hash proves identity, not completeness. An editor-only importer/helper reference leaking into runtime packages is a release failure. Remove unnecessary import metadata after checking its role and revalidate dependency closure.

To inspect a cooked PAK, `UnrealPak <pak> -Extract <dir>` writes paths relative to the PAK's mount point, which `-List` reports. Name tables show property names only when values are non-default; string presence is a clue, not proof.

## Installation

Install only into a confirmed game installation selected for testing. Wait for the game and server to close; do not terminate another user's session automatically. Back up the specific prior mod PAK, copy the verified replacement, and compare installed/build SHA256. Leave unrelated PAKs alone.

Record the installed version and rollback artifact privately. Test loading and intended behavior in the shipping game; archive integrity cannot establish runtime compatibility.

## How Unchained treats hand-installed PAKs

Source inspection of Unchained Launcher 1.0.2, UnchainedPlugin 1.0.0-rc4 and Unchained-Mods 0.2.2 / 0.2.3 (November 2026), plus the in-game observations marked below. Check the versions your players run.

- **Launcher prompt:** at launch the launcher hashes every `*.pak` in the `Paks` root except `pakchunk0-WindowsNoEditor.pak` and asks about each PAK it does not manage ("Unmanaged Mods detected"). **Keep** is remembered per file name and hash, so every rebuild asks again; **Remove** deletes the `.pak` and its `.sig`; **Cancel** aborts the launch. The `always_keep_unmanaged_paks` setting skips the prompt.
- **Duplicates:** byte-identical PAKs in the root are deleted without asking, together with their `.sig`, unless the `.sig` files differ. Keep one copy of each build.
- **Adoption:** choosing Keep on a PAK whose hash matches a registered release makes it launcher-managed.
- **Do not park PAKs in `Saved/Paks` folders** (`TBL/Saved/Paks`, `%LOCALAPPDATA%/Chivalry 2/Saved/Paks` or `Saved_<suffix>/Paks`): the launcher moves `.pak` and `.sig` files from there into the game's `Paks` folder.
- **Subfolders of `Paks`:** UE 4.25 mounts PAKs recursively, and library PAKs in a subfolder loaded in game. The plugin registers every mounted PAK, subfolders included. The launcher never prompts about, deletes or scans subfolder PAKs, so their mod actors and maps do not appear in its server configuration; keep map PAKs in the root.
- **Names and mounting:** the plugin drops PAKs named `pakchunk*` from mod discovery and excludes a PAK that fails to mount (for example a bad `.sig`; its log then says `mount failed (excluded)`).
- **Discovery cost:** a mod PAK may carry a `ModAssetRegistry.bin` under its `Content/Mods/` folder, which the plugin merges directly. Without one, the plugin parses the header of every `.uasset` in that PAK once at startup, which matters for large libraries.
- **Shown version:** the `ModVersion` of a `DA_ModMarker_V2`, else the mod actor's `ModVersion`, else "Unknown".
- **Paths Unchained-Mods replaces:** `CampaignFrontendTracker`, `LoadingScreensConfig` and `AIPersonalities` under `/Game`. Mod PAKs must not ship those paths, nor copies of `/Game/Mods/ArgonSDK/**`.
- **Load delay (in game):** Unchained-Mods 0.2.0 rescanned `/Game/Mods` synchronously on every map load, so the delay grew with the number of installed mod assets (about 32 s with a large library). Version 0.2.2 rescans only when nothing is registered yet; the same setup then loaded in 1-2 s.
- **Main-menu map lists (Unchained-Mods 0.2.3, release notes and asset names, not yet checked in the menu):** the map Options box and the typed map name field are gone. Map dropdowns are built from `DA_MapGroupInfo` data assets (`/Game/Mods/ArgonSDK/Mods/ModLoading/v2/DataAssets/`). Each group has a display name, an explicit map list and wildcard patterns matched against the map short name, ignoring case. The shipped groups include the official modes (`FFA_*`, `TO_*` and so on) plus Chivalry 1 (`*_CMW_*`), Battle Royale (`*_BR`) and King of the Hill (`*_KOTH`). Name custom maps with the mode prefix so they land in that mode's list. A map can appear in several groups. To add a category, ship your own `DA_MapGroupInfo` asset in your mod folder; groups are additive.

## Registry publishing format (v1)

Documentation and source inspection, October 2026: see [C2ModRegistry](https://github.com/Chiv2-Community/C2ModRegistry), the [mod setup docs](https://github.com/Chiv2-Community/zero-infra-mod-registry/blob/v1/docs/mod/README.md) and schema `schemas/mod.schema.json`, and the optional [ModTemplate](https://github.com/Knutschbert/ModTemplate). Older publishing guides predate this format.

- `mod.json` sits at the repository root. Required: `name`, `description`, `authors` (list). Optional: `$schema`, `repo_url`, `icon_url`, `image_urls`, `mod_dependencies`, `plugin_dependencies`. A dependency is `{repo_url, version}`, where `version` is a SemVer, caret range or comma-separated comparisons (`>=0.9.0,<1.0.0`). Legacy `dependencies` still reads but is deprecated; extra fields are allowed. A `mod.json` release asset overrides the committed one.
- A release needs a semantic-version tag (`v0.9.0`), must not be a pre-release, and carries exactly one `.pak`, which the registry scans. A set of many PAKs cannot ship as one release.
- Register each version through an issue (template *Add Release to Registry*, JSON body with `"action": "add-package-release"`, `repo_url` and `release_tag`) or an automated workflow; reply `retry` after fixing a failure. Every version needs maintainer approval, and dependencies must already be registered. Releases created by a workflow's default token do not trigger other workflows.
- Launcher 1.0.0-RC4 read only the legacy `dependencies` field; entries carrying only `mod_dependencies` lost their description and crashed at launch until the registry side changed. Check a registered entry in the launcher version your players use.
