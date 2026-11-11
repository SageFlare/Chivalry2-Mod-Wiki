# Converting Chivalry: Medieval Warfare content

Reusable UE3-to-UE4 conversion lessons from bounded experiments. **This wiki does not supply a complete automatic CMW-to-Chivalry-2 converter.** Existing findings describe algorithms and validation requirements without private parser or editor-helper dependencies.

| Guide | Scope |
| --- | --- |
| [Collection and stage contracts](workflow.md) | Source identities, immutable bundles and explicit conversion boundaries |
| [Geometry and collision](geometry-and-collision.md) | Transforms, attributes, terrain, hulls and route checks |
| [Materials and environment](materials-and-environment.md) | Shaders, baked lighting, foliage, effects, audio and state |
| [Tools and evidence](tools-and-evidence.md) | Public tool roles and limits of claimed support |

Begin with a legally available source map and its referenced packages. A lone cooked `.udk` is often insufficient. Preserve texture caches and streaming levels. Prefer authored uncooked source when available, but verify that it matches the desired map version.

The observed PC serialization included package version 860, licensee 0, engine 10246. That tuple alone does not certify parser support for compression or every native payload.

Keep extraction and original source read-only. Build a new target project/artifact. Original objective logic, gameplay, and new C2 replacements need explicit independent scope; recovering scenery does not translate a Team Objective mode.
