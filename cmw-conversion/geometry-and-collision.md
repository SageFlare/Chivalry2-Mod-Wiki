# Geometry, attributes and collision

## Static meshes and LODs

- Prefer an import route that keeps topology, material sections, all UV sets, colours, normals and tangents. OBJ loses extra UVs and colours. The UE 4.25 bundled Datasmith glTF importer supported an assets-only import without a dialog (the basic glTF importer always showed one). Disable generated light-map UVs and unrelated imports.
- The Datasmith glTF path stored linear vertex RGB as sRGB-encoded bytes. Pre-converting a working copy's RGB with inverse sRGB (alpha unchanged) recovered source bytes within quantization. Prove this for your accessor format first.
- Datasmith names material slots `0`, `1`, ... When appending LODs, rename slots and their polygon-group slot names uniquely. Duplicate names corrupted face slots, LOD0 included.
- UModel (build 1590) writes LOD0 only, even with `-lods`. Higher LODs need a native decoder, checked bit-exact against UModel's LOD0. Map per-LOD slots by material, not element index.
- UModel static-mesh exports were not deterministic between identical runs. glTF `COLOR_0` appeared or not, the tangent w sign flipped, and the PSKX vertex-colour chunk changed. Positions, normals, UVs, indices and slots stayed equal. Do not take UModel vertex colours for version-860 meshes. Compare UModel meshes with those fields masked.
- Exported non-white colours are not proof of source colours. Compare against native buffers. Keep component paint separate from mesh colours.
- Keep zero-triangle sections as empty slots so slot indices stay the source's.
- Replace characters UE4 refuses in asset names (such as spaces) per path segment, and assert one source identity per resulting path.

## Lost channels and correspondence

- To retrofit a lost UV channel, match triangles one-to-one by positions, existing UVs, winding and section. Overlapping opposite-facing triangles can share positions and UV0 yet have different light-map UVs. Use cyclic corner permutations and reject ambiguity.
- The UE 4.25 glTF importer drops a third UV channel. A light map stored in UV channel 2 had to be written natively onto the imported mesh. Check before and after that only that channel changed.
- Compare saved meshes independently: positions, triangle mapping, UVs, colours, tangents, sections, bounds. A successful import is not an equivalence check.

## Compiled BSP

Audit compiled BSP separately from actors.

- Nodes are not always planar (up to about 3 units off their stored plane). Keep them as stored. An exact match against the render buffers is the check, not planarity.
- Compute expected triangle counts with UE4's own FBX degenerate test (squared cross-product length at most 1e-8 in float32 is dropped), not a fitted area threshold.

Evidence: editor.

## Terrain

- Rebuild terrain as static sections from the real height data, dimensions, spacing, section offsets and scale. Constants from one map do not transfer.
- Reproduce CMW's `LandscapeVertexFactory` coordinate math, including two-subsection components. Rebuild normals per pixel from the heightmap's packed B/A channels.
- Translate the layer material with UE 4.25 `LandscapeLayerBlend` semantics. Layer names are the static terrain-layer parameters. Omitted instance masks default to alpha. Unallocated layers must be zero.
- Weight maps that need alpha must not go through PNG. UModel drops a constant alpha channel. UE 4.25 PNG import also overwrites the RGB of alpha-0 pixels. Write the source top mip as an uncompressed DDS instead.

Evidence: editor and in game.

## Skeletal props

Non-animated skeletal mesh actors became static copies of the bind pose with the mesh `Origin` and `RotOrigin` baked in, since UE3 applies `RotOrigin` at render time. Read the mesh header natively; these fields are not tagged. Blocking ones use complex-as-simple collision.

## Transforms

Compose actor, component and instance transforms, including mirrored scale. Compare saved values with float32-aware tolerances. Scale tolerance grows with magnitude. Rotations need a looser tolerance near ±90° pitch because spawn and read-back pass through quaternion conversions.

## Collision

- Collect mesh body setup, blocking volumes, component overrides and effective query policies separately. A visible mesh may be non-blocking by inherited default.
- **Mesh rule:** use the source convex hulls when the export has hulls, no primitive shapes and all `UseSimple*` flags hold (unserialized means true). Otherwise use complex-as-simple. Component collision still decides per placement.
- UE 4.25 Python: set a named profile with `set_collision_profile_name` only. Setting the object type, or any response that changes a value, turns the profile into `Custom`.
- **Volumes** (blocking, kill, pain, reverb) use exact UE4 classes and exact source hulls. UE4 containment is exact (`EncompassesPoint` distance 0). Large rectangular hulls need analytic box elements: as convex elements they left points inside counted as outside. Verify containment with dense probes on the source planes, not hull centres.
- Read a source flag before assuming behaviour. CMW's water volume class never sets `bWaterVolume`, so there is no swimming. Kill-on-touch water mapped to `KillZVolume` is recorded as an approximation.
- Chivalry 2 picks footstep sounds from the surface physical material, while CMW used a volume trace. Floors inside source water volumes need a water physical material on the component override and on each placement.

## Helper geometry

Generated helpers (query hulls, markers) must be inert to Chivalry 2 gameplay. Make them query-only, give them their own object type and set **every channel to Ignore**. A helper that overlapped the `Weapon` channel made kicks and tackles stop instantly (in game). Compare new collision against the retail `DefaultEngine.ini` profiles.

Full-scene scaling is a completion rule: [completion rules](completion-rules.md#scale).
