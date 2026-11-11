# Editor automation in UE 4.25.4

The following are observations from the tested ArgonSDK environment (editor and commandlet runs unless stated otherwise). Probe APIs against your installation; newer Unreal Python documentation may describe unavailable methods. Before a long run, climb the [preflight checks](preflight-checks.md).

| Task | Launch mode |
| --- | --- |
| Read-only inspection of saved assets, loading maps | `UE4Editor-Cmd.exe <absolute-project> -run=pythonscript -script=<script> -NullRHI -ForceDependsGathering` |
| Place or spawn actors, create levels | Full `UE4Editor.exe`. Headless `-NullRHI` sessions crashed in `new_level` and actor spawning, while loading maps and assets worked |
| One-pass script | `-ExecutePythonScript=<script>`; the editor exits after the script. Do not call `quit_editor()` (shutdown crash after saving) and do not add `-ExecCmds="QUIT_EDITOR"` (it runs first, so the script never runs) |
| Wait for registry, navigation or render ticks | `-ExecCmds="py <script>"` with a Slate tick callback; the script calls `quit_editor()` itself |
| Many short steps on one large map | A resident editor (below) |
| Bake lighting | See [lighting](../visuals-and-audio/lighting-and-materials.md) |
| Cook | See [packaging](../packaging-and-hosting/cook-and-package.md) |

Use absolute project and log paths; a relative project path crashed before any log was written. Useful flags include `-EnablePlugins=EditorScriptingUtilities -unattended -nop4 -nosplash -abslog=<log>`. UE 4.25 reads only the first `-EnablePlugins=` on a command line (source inspection), so put every plugin in one comma-separated list, for example `-EnablePlugins=EditorScriptingUtilities,SequencerScripting` for a run that edits level sequences. Check exit status and a script-specific completion marker; a missing marker is a failure. Run one authoring editor per project; do not interrupt an existing user session.

Editor caches and build tooling need normal writable cache locations (shader working directory, Boot.ddc, the build tool's configuration cache). Cache permission failures can resemble asset failures. Diagnose the failing path before changing content.

## Launching without stealing focus

A process flag that hides console windows (such as `CREATE_NO_WINDOW`) does not hide a GUI editor's window or splash screen; the editor still appeared and took focus from a running game. Launch GUI editors with `STARTUPINFO` using `STARTF_USESHOWWINDOW` and `wShowWindow = 0` (SW_HIDE), and pass `-nosplash`. Keep a step timeout as the backstop for hidden crashes. Focus also matters for probes: a focused editor ran at a very different frame rate than a background one.

## Waiting safely

Wait for the asset registry before authoring. Guard callbacks against re-entry, since loading and saving pump Slate, and unregister a one-shot callback before saving. Load maps when the script starts, not inside a registered tick callback: loading there and continuing to tick crashed the editor. Asynchronous work needs a bounded state machine that waits for completion before saving and exiting; script return does not prove navigation or rendering has finished.

Load needed assets before entering simulate mode: `load_asset` returns nothing while the editor plays.

## Resident editor for many short steps

Restarting the editor and reloading a large map per step can dominate run time. One process can serve a whole session:

- Start it with `-ExecCmds="py <server>" -RUNNINGUNATTENDEDSCRIPT`. `-ExecutePythonScript` quits on the tick after its script returns.
- The tick callback only detects a queued job. Run the job itself through `DEFER py <dispatcher>`, the same context as `-ExecCmds`; running jobs inside the callback risks the map-load crash above.
- Per job, set and clear environment variables in both Python's `os.environ` and the Win32 process environment (native code reads the latter), drop cached workspace modules, and run the script as `__main__`.
- Native plugins that cache environment or file values in `static` variables must read them per call when they share a process.
- Write a unique end line per job and read the log until that line is on disk. File size reported while the editor holds the log open, or a log flush command, did not guarantee the job's lines were written.
- Restart for a cold reload when switching between authoring and verification, when a job leaves dirty packages, before host steps that must not run beside an editor, and after any failure.
- Keep headless validations in a separate `-run=pythonscript -NullRHI` process.

## API pitfalls

| Operation | Tested rule |
| --- | --- |
| Python version | Python 2.7.14: no f-strings or `exist_ok`; host Python 3 checks prove nothing |
| Blueprint class | `EditorAssetLibrary.load_blueprint_class(path)` |
| Blueprint variables | Use raw names, including spaces (`'Max Score'`); there is no snake_case alias |
| Rotation | `Rotator(pitch=..., yaw=..., roll=...)`; positional order is roll, pitch, yaw. No `Rotator.quaternion()` or `rotate_vector`; use `Vector.rotate(rotator)` and compare axis vectors |
| Enums | Compare with the enum member; `str()` prints `<ComponentMobility.STATIC: 0>` |
| Copy actor | Spawn by class and copy intended properties; `duplicate_actor` is absent |
| Re-run a construction script | Make a real change; setting the current value does nothing. Re-fetch components afterwards |
| Discover maps | AssetRegistry; `EditorAssetLibrary.list_assets` omits map packages, and maps cannot be deleted or renamed through it |
| Fresh dependencies | `scan_paths_synchronous([folder], force_rescan=True)`; reject a None result |
| Material parameter setters | Read back values; a false return was observed after successful changes |
| Compare vectors | Compare numeric components, not object strings |
| Arrays | `unreal.Array + <empty unreal.Array>` crashed the editor; concatenate Python lists |
| Mesh sections | `mesh.get_num_sections(lod)`; `EditorStaticMeshLibrary` has none |
| Capture export | Call `capture_scene()`, then allow render ticks before exporting |
| Simulated actor function | `execute_console_command(world, 'ke <full actor path> <Function>')`; a bare name hit the editor-world copy |
| Runtime actor destruction | `destroy_actor()`; exposed Python names can omit K2 prefixes |
| Level sequences | Enable SequencerScripting for that run; read a placed actor's sequence with `load_sequence()`, since `get_sequence()` returned None in a cold commandlet |
| Gameplay tags | `GameplayTagLibrary` (not `BlueprintGameplayTagLibrary`); structs have no `export_text()` |
| Kismet libraries | Named `SystemLibrary`, `StringLibrary`, `MathLibrary` |

Absent in this Python API: Blueprint graph editing, `Object.call_method`, `new_object`, `editor_end_play`, deferred spawning under Simulate, `SceneCapture.HiddenComponents`, Cascade particle module classes, reading or listing material expressions and their editor positions, and the mesh `bAutoComputeLODScreenSize` property. Use the editor UI, T3D export, or a small editor plugin. Do not assume an unavailable method succeeds because a host-Python syntax check passes.

## Saved state

Keep authoring and validation separate, and validate saved output in a fresh process. Partial files may survive exceptions (for example a level written before a script failed); identify exactly what the failed run created before repairing or retrying.

Saving a map that has build data also re-saves `<Map>_BuiltData` with a new package GUID. Back up and restore both files together, and do not expect byte-identical build data after any map save. `save_map(world, new_path)` on a map that already exists on disk writes a copy while the editor stays on the original; load the new path and assert the current world's package name before editing.

Many SDK functions are empty stubs; check the SDK source before relying on an editor-time call.

For the tested plugin stack, VS2019 avoided compiler failures encountered with newer MSVC. Content-only project plugin builds targeted the engine editor target. These are version-specific toolchain observations, not instructions to change every machine's global build configuration.
