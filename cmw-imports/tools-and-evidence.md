# Tools, verification and evidence

## Public tools

Pin the versions you use. Advertised format support never establishes extraction of every native array, landscape, collision body or shader graph.

| Resource | Role that worked | Limit found |
| --- | --- | --- |
| [UModel / UE Viewer](https://www.gildor.org/en/projects/umodel) (build 1590) | Mesh LOD0 (glTF, PSK/PSKX slots), PNG/DDS textures, material metadata, sounds | LOD0 only, even with `-lods`. Vertex colours and tangent signs not deterministic. Drops a constant alpha channel. `-sounds` needs `-groups`. Not long-path aware: past 260 characters it logs `Error creating file` but still exits 0. No ShadowMap2D subclass |
| [UELib](https://github.com/EliotVU/Unreal-Library) | Package properties, static actor decoding, bounded native arrays | No texture-cache (`.tfc`) support; CMW listing is not full map/terrain/foliage extraction |
| [UE Explorer](https://github.com/UE-Explorer/UE-Explorer) | Inspect properties, defaults and UnrealScript | Inspection, not conversion |
| [UT4X-Converter](https://github.com/xtremexp/UT4X-Converter) | Scene-text translation after adaptation | CMW and Chivalry 2 are not supported targets; check its licence terms |
| [UDK-to-UE4 T3D converter](https://github.com/GameSMRT/UDK-to-UE4-T3D-Converter) | Text scene conversion | Tested output had duplicate records and invalid scale; rejected |
| CMW SDK editor, shipped UnrealScript and shaders | Native T3D export, class defaults, hulls, renderer semantics | Declarations are not native C++; exports omit some native payloads and default values |
| [LegendaryExplorer](https://github.com/ME3Tweaks/LegendaryExplorer) UDK serializers | Reference layout for native component data | Cross-check only, not a converter |
| UE 4.25 bundled Datasmith glTF importer | Attribute-preserving mesh import | Colour encoding, slot names, UV channel 2 |
| RenderDoc on the offline CMW SDK | Runtime constants and attribute checks | One draw does not prove every component or state |

See the [CMW SDK documentation](https://tornbannerjira.atlassian.net/wiki/spaces/CHIVCOM/pages/5013637/Chivalry+SDK). Many steps also needed custom native readers for CMW package data, and native editor code for engine API that UE 4.25 Python hides. Examples: material output connections, Cascade modules, build data. None of that is supplied here.

## Verification levels

1. **Static:** the plan or translator agrees with source data, run host-side before any editor session. Run every later stage's planner against the current inputs; it finds most data errors in minutes instead of a failed hour-long run.
2. **Editor:** reopen saved output in a separate process and check counts, transforms, attributes, references, collision, material compile results and usage flags. Fail a session on compile or usage-flag errors that name your content.
3. **Rendered evidence:** capture each converted effect system and keep the images. Look at them first when the game shows a broken effect.
4. **Package:** cook the explicit map, check the archive's content and dependency closure, then compare installed and built hashes.
5. **In game:** a person checks named features. Record their verdict against the build identity, scoped to what they looked at.

A final pass re-runs every stage's verification against the finished content. Coverage gives every scene actor a disposition: converted, approximated, unsupported, excluded or unverified. A validator that skips missing rows passes on an empty set, so require every row of an authored family.

## Failure handling

- **Never substitute silently.** Every missing, approximate or unsupported item goes into a report with its source identity.
- Retry a stage once from a clean state only when it ended with no result and no error (a crash).
- An optional stage (effects, audio, decals, light volume) that still fails may be skipped only if content and editor state equal the stage's start. The map is then partial and is not installed.
- Item failures (one effect system, one cue) may be set aside and listed when the failure names the item. Fail the whole stage when set-aside items exceed a systemic threshold (we used 25 items or 10 %). A failure that names no item fails the stage.
- **Reuse:** re-run only the longest unchanged prefix of stages, keyed by code, plugin builds, inputs and a content hash of each plan (hash JSON by content, since key order can vary). Keep stage outputs out of plan folders. Do not edit code while a run uses it.
- The editor can need a lot of memory. Placing about 1,000 static-mesh rows committed about 7.5 GB in one session. The limit is commit charge (RAM plus page file), and mid-session garbage collection did not lower it. Split placement into sessions instead.

## In-game crash triage (no instrumentation)

- Unchained writes `unchained.log` (map loads, match states, mod loader step, access violations) and crash folders under `%LOCALAPPDATA%\Chivalry 2\Saved_Unchained\`. The shipping build has no symbols; frame names there are nearest-export guesses.
- Narrow statically: list features that occur on crashing maps and on no working map.
- Bisect without the editor: extract the built archive, repack it without one package or folder group, test, list, install and compare hashes. Keep the original archive as the restore point.
- A map can hold two culprits. A bisect that removes one candidate per round from the full archive cannot clear it. Group suspects by shared feature first, then split.
- For a property question, build tiny probe maps with one variant each. The editor did not reproduce a game-only crash, so probe maps were the test.

Do not attach to or modify a running game for this. Instrumentation is a separate decision with its own risks.
