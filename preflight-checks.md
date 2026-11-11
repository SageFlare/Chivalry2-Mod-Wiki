# Preflight checks before editor runs and cooks

A prepared editor run, rebuild or cook can take 10–20 minutes, and a failure usually adds a rollback and a rerun on top. In one recorded batch of map rebuilds (UE 4.25.4 ArgonSDK), most of the failed run loops could have been caught by a check costing seconds or minutes. Climb the ladder below cheapest first and stop at the first failure. Skip a rung only with a stated reason, such as "no native code changed".

## The ladder

1. **Static checks (seconds, no editor).**
   - Lint editor Python against the engine's generated API stub (`<ProjectRoot>/Intermediate/PythonStub/unreal.py`, about 20 MB in the tested setup). The stub reflects the enabled plugins, so regenerate it after any plugin change. A lint of `unreal.<Class>.<member>` references catches calls that do not exist in 4.25 (for example mesh-LOD build-settings methods that arrived in 4.26) and wrong argument names. Instance methods on wrapper values (`some_rotator.quaternion()`) can slip past a simple class-member lint.
   - Check syntax for the right interpreter: Python 2.7 for editor scripts, Python 3 for host scripts. Flag list comprehensions that reuse a live variable name; in Python 2.7 the loop variable leaks and silently replaces the earlier value.
   - Lint Windows PowerShell 5.1 runners: variables differing only by case (PowerShell names are case-insensitive); `"$n: text"` (parsed as a drive reference, so the whole script fails to load: write `"${n}: text"`); `@(... | ConvertFrom-Json)` on a JSON array (wraps the array as one element: assign first, then enumerate); assignments to `$args`/`$input`; functions named like built-in aliases. Parse-check a new runner before using it.
2. **Compile native code (minutes).** Build changed editor plugins during preparation into a fresh, short output folder. UnrealBuildTool fails on paths over 260 characters, and the engine build treats some warnings (for example C4456, shadowed declarations) as errors.
3. **Data checks against engine limits (host).** Check prepared inputs against the table below, for example "this glTF uses a third UV channel" or "this extent is not exact in float32". Add every newly found limit to your own table.
4. **Canary run (editor, minutes).** Run the whole prepared chain on one or two representative assets in a scratch copy. Pick unusual cases: rotated props, meshes with extra UVs or vertex colours, multi-section meshes, the largest material. Run only behaviour checks: completion markers, end-state asserts, validators. A canary catches what reading cannot, such as cook behaviour, rebuild side effects and vertex splits.
5. **Independent review.** Give a reviewer the results of rungs 1–4. Every assumption the plan marks "checked at run time" must have been exercised by the canary.
6. **Full run.** Snapshot, apply, verify, validate saved state in a fresh process, then cook and package. Test the result in the shipping game separately.

Before changing an asset's shape (material slots, LODs, UV count), find every validator and build gate that reads that property.

## UE 4.25 behaviours that broke runs

Evidence: **editor** = observed in editor/commandlet runs; **source** = engine source inspection; **cook** = cook output; **game** = shipping game. Python-API gaps are listed in [editor automation](editor-automation.md).

| Behaviour | Consequence | Workaround | Evidence |
| --- | --- | --- | --- |
| A material used on a mesh type it lacks a usage flag for (for example instanced static meshes) gets the flag only in editor memory; the cook does not warn | Grey checker material in game | Set the usage with `MaterialEditingLibrary.set_material_usage` at import and save; fail any later session that logs `needed to have new flag set` or `missing the usage flag` | game |
| `unreal.find_object` follows redirectors by default; `does_asset_exist` is true for a redirector and `load_asset` follows it | Checks see the destination asset | Pass `follow_redirectors=False`; keep your own alias list | editor |
| `consolidate_assets` rewrites hard references only; `FixupReferencers` is not exposed to Python | Soft references keep old paths | Keep redirectors; check hard imports | editor |
| The cooker skips redirector-only packages and rewrites cooked soft paths to the destination | "Every content package is in the PAK" audits fail | Exclude known redirectors | cook |
| Re-saving a material instance refreshes its cached referenced textures from the parent | Unexpected diffs | Allow entries already present in the parent | editor |
| PNG import replaces the RGB of alpha-0 pixels | Data textures lose colour | Import such data as BGRA8 DDS | source |
| The texture importer may classify bluish images as normal maps; sRGB does not stick while compression is the normal-map setting | Some colour textures stay linear | Set LOD group, then compression, then sRGB; read sRGB back | editor |
| The glTF importer reads `TEXCOORD_0/1` only | A third UV channel is dropped silently | Count UV channels before import | editor |
| Static-mesh UVs are half precision unless full-precision UVs are enabled | Large-range coordinates derived from UVs drift | Derive them from world position | editor |
| FBX/OBJ import computes normals by default; an OBJ without smoothing groups shades faceted | Curved generated geometry looks faceted | Write normals and import them | editor |
| Recomputed normals/tangents split vertices at seams; stored tangents of FBX/OBJ meshes can be zero | "Render vertices ≤ source" and tangent asserts fail | Bound by 3 × triangles; read the build settings | editor |
| All-white vertex colours are dropped on save | The colour stream disappears | Treat all-white as no stream | editor |
| `SetStaticMesh` returns false when nothing changed | False failures | Assert the end state | editor |
| Rotations read back near pitch ±90° differ slightly | Exact rotator compares fail | Compare axis vectors with tolerance (1e-4) | editor |
| Float properties are float32; above about 1.3e5 units the step exceeds 0.01 | Tight readback checks fail | Plan float32-exact values or use relative tolerance | editor |
| T3D export omits struct members equal to the struct default | Parsers read missing members as zero | Fill omitted members from the struct defaults | editor |
| Custom primitive data holds 32 floats; `SetCustomPrimitiveData*` writes a transient copy; a material expression's data index is fixed | Values exceed space or are not saved | Set the saved `custom_primitive_data` property; encode choices in the material instance | editor, source |
| Texture2DArray creation is gated by an experimental setting | Array textures may not author in automation | Use 2D atlas pages | source |
| Disabling a cull-distance volume only flags the world for later | Components keep cached draw distances when saved straight away | Force each component's property-change refresh; assert the cached value before saving | source, editor |
| Once a level holds a precomputed light volume, unbuilt static primitives use the indirect lighting cache unless its quality is off | Volume lighting also tints static meshes | Turn the cache off on static primitives | source |
| Exponential height fog stores density and falloff at artist scale (the renderer divides by 1000); each bloom stage tint is scaled by 1/6 | Values copied in renderer units look 1000× thinner or 6× weaker | Convert to the stored scale | source |
| Lightmass threads come from the Swarm agent's `LocalJobsDefaultProcessorCount` (per-user `UnrealEngine/4.25/Saved/Swarm/SwarmAgent.DeveloperOptions.xml`); the editor's unused-cores option had no effect | Thread limits set in the editor do nothing | Edit that setting before Swarm starts and restore it; read the thread count from the Swarm job log | editor |
| A custom commandlet that sets `IsClient`/`IsServer` false runs without a client | Access violation after a lighting build | Keep the commandlet defaults | editor |
| Call-function nodes are keyed by the C++ function name, not the display name; `GetActorOfClass` is impure; `GetIsReplicated` is not a UFUNCTION | Missing nodes or pruned calls | Check the 4.25 header; wire impure calls into the exec chain | editor |
| Exported node text writes floats as `0.000000` and bools as `true`; break-struct outputs share pin names; linked inputs still carry a default | Text checks fail correct graphs or pass wrong ones | Parse the input pin's own line and compare numerically | editor |
| `RunUAT.bat` starts `AutomationTool.exe` by bare name | Exit 9009 when `NoDefaultCurrentDirectoryInExePath` is set | Clear that variable for the build process | editor |
