# Public tools and evidence boundaries

These roles summarize historical research and bounded experiments, not a newly verified compatibility matrix. Pin the versions used for your own work.

| Resource | Useful role | Limit |
| --- | --- | --- |
| [UModel / UE Viewer](https://www.gildor.org/en/projects/umodel) | Meshes, textures and material metadata | Does not reconstruct a complete map scene; exporter attributes require verification |
| [UELib](https://github.com/EliotVU/Unreal-Library) | Package properties and script/native-data investigation | Declared CMW support is not complete terrain, foliage or texture-cache extraction |
| [UE Explorer](https://github.com/UE-Explorer/UE-Explorer) | Inspect package properties and UnrealScript | Inspection/decompilation is not full map conversion |
| [UT4X-Converter](https://github.com/xtremexp/UT4X-Converter) | UE3/UDK conversion architecture and scene translation | CMW/C2 are not established drop-in targets; review its own terms and supported routes |
| [UDK-to-UE4 T3D converter](https://github.com/GameSMRT/UDK-to-UE4-T3D-Converter) | Text scene conversion | Tested output had duplicate-record/scale issues; validate before use |
| CMW SDK editor and shipped UnrealScript/shaders | Native scene export, class defaults and renderer semantics | Declarations are not the missing native C++ implementation; exports can omit native payloads |
| UE 4.25 bundled Datasmith glTF importer | Attribute-preserving mesh import candidate | Verify color encoding, sections, tangents, LODs and runtime dependency cleanup |
| RenderDoc | Attribute/runtime-constant observations in a compatible offline source environment | A sampled draw does not prove every component/state or shipping-target compatibility |

The [CMW SDK documentation](https://tornbannerjira.atlassian.net/wiki/spaces/CHIVCOM/pages/5013637/Chivalry+SDK) is a starting point for source content and native workflows. Inspect CMW-specific methods before assuming a generic UE3 tool is sufficient.

A successful static actor decode proves only that scope. A tool's advertised format support does not establish extraction of every native array, landscape, collision body or shader graph. Keep unsupported data and compare independent routes where practical.

Record exact commands, versions, source/tool hashes, input identities and output inventories in your own evidence bundle. Shared records should use relative references and omit personal paths. Never invent missing provenance for an older export.

When an exporter rejects a subclass, a disposable-copy experiment may help establish inherited storage compatibility. Such a workaround must preserve originals and be checked against independent pixel/structure evidence; do not describe it as direct supported extraction or patch the installed source packages.
