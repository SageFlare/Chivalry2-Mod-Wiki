# Editor automation in UE 4.25.4

The following are observations from the tested ArgonSDK environment. Probe APIs against your installation; newer Unreal Python documentation may describe unavailable methods.

| Task | Launch mode |
| --- | --- |
| Read-only inspection of saved assets | `UE4Editor-Cmd.exe <absolute-project> -run=pythonscript -script=<script> -NullRHI -ForceDependsGathering` |
| Place actors or author a level | Full `UE4Editor.exe` |
| One-pass script | `-ExecutePythonScript=<script>`; let the editor exit automatically |
| Wait for registry, navigation or render ticks | `-ExecCmds="py <script>"` with a Slate tick callback and explicit exit |
| Bake lighting | See [lighting](lighting-and-materials.md) |
| Cook | See [packaging](cook-and-package.md) |

Use absolute project and log paths. Useful flags include `-EnablePlugins=EditorScriptingUtilities -unattended -nop4 -nosplash -abslog=<log>`. Check exit status and a script-specific completion marker. Run one authoring editor per project; do not interrupt an existing user session.

Editor caches and build tooling need normal writable cache locations. Cache permission failures can resemble asset failures. Diagnose the failing path before changing content.

## Waiting safely

Wait for the asset registry before authoring. Guard callbacks against re-entry: loading and saving can pump Slate. Unregister a one-shot callback before saving. Asynchronous work needs a bounded state machine that waits for completion before saving and exiting; script return does not prove navigation or rendering has finished.

For reproducible probes, record frame count and elapsed time. Foreground/background focus can change editor tick rate substantially. Load needed assets before entering simulate mode: asset loading through editor libraries may refuse during play.

## API pitfalls

| Operation | Tested rule |
| --- | --- |
| Python version | Bundled environment was Python 2.7.14: no f-strings or `exist_ok` |
| Blueprint class | `EditorAssetLibrary.load_blueprint_class(path)` |
| Rotation | Use `Rotator(pitch=..., yaw=..., roll=...)`; positional arguments have unexpected order |
| Copy actor | Spawn by class and copy intended properties; do not assume `duplicate_actor` exists |
| Discover maps | Use AssetRegistry; `EditorAssetLibrary.list_assets` omits map packages |
| Fresh dependencies | Force a synchronous registry scan and reject missing results |
| Material parameter setters | Read back values; a false return was observed even after successful changes |
| Compare vectors | Compare numeric components, not object strings |
| Capture export | Call `capture_scene()`, then allow render ticks before exporting |
| Runtime actor destruction | `destroy_actor()`; exposed Python names can omit K2 prefixes |

Blueprint graph editing, `Object.call_method`, and several newer editor convenience methods are absent in this Python API. Use the editor UI or separately evaluated public tooling. Do not assume an unavailable method succeeds because a host-Python syntax check passes.

Keep authoring and validation separate. Validate saved output in a fresh process. Partial files may survive exceptions; identify exactly what the failed run created before repairing or retrying.

For the tested plugin stack, VS2019 avoided compiler failures encountered with newer MSVC. Content-only project plugin builds targeted the engine editor target. These are version-specific toolchain observations, not instructions to change every machine's global build configuration.
