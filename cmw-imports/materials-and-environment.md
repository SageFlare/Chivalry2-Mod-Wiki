# Materials and environment

## Material graphs

- Resolve a material by the path its user saves (component slot, mesh element import path, or the mesh's own package), never by object name across packages.
- **Function calls:** inline UE3 material function calls. Bind inputs by Id and cast each input to its declared type (Scalar takes `.r`, as UE4's `ValidCast`). The call's output is the FunctionOutput's A input. Texture-object and static-bool arguments bind by reference. An unconnected function input needs `bUsePreviewValueAsDefault`.
- **Unconnected inputs:** an exported input without `Expression=` is unconnected. A static switch that selects an empty branch counts as unconnected too.
- **Node translation:**
  - A dynamic `If` maps to UE4 `If` only when its equal branch repeats another branch.
  - UE3 `CameraVector` is tangent-space: use `CameraVectorWS` plus a world-to-tangent transform.
  - `CameraWorldPosition` becomes `CameraPositionWS`; `DepthBiasedAlpha` becomes `DepthFade`.
  - `SceneDepth` translates only without normalization and without connected coordinates.
  - Distortion has no route and is a listed drop.
- **Water normals:** an Unlit water material ignored its normal input. Feed the normalized source normal into the reflection and Fresnel consumers explicitly.
- **Unbound samplers:** a source material whose output reads a texture sampler without a texture does not compile in the CMW SDK. Record it; never invent a binding.
- **No silent defaults:** stop on a material with nothing connected, or a slot with no material, unless an explicit deferral names it with a reason. List every dropped channel.
- **Usage flags:** save `bUsedWithInstancedStaticMeshes` at import. Placing foliage set it only in memory, and the cooked game drew the default material (in game). Sprite and mesh-particle flags likewise. See [lighting and materials](../lighting-and-materials.md).
- **Textures:** set group, then compression, then sRGB. The importer can mistake a light map for a normal map.

Function-call and water translations: editor and cook; in game for some graphs only.

## Decals

UE4 deferred decals cannot draw on Unlit surfaces. Ported static decals became **receiver geometry**:

- Clip each receiver's front-facing triangles to the decal frustum (near to far along the negative hit normal, ±W/2 and ±H/2 across). Take receivers from the serialized `DecalReceivers` list; component names repeat across actors.
- u = 0.5 − t/W, v = 0.5 − b/H. Drop slivers with an edge under 0.05 units.
- Light each piece with its receiver's light map through a decal-material adapter; for vertex-lit receivers, interpolate the lighting coefficients at the clip points. Unlit decal materials need no light data. Pieces on movable receivers attach to the receiver.

Approximations: the decal normal map is not rotated into the receiver's tangent basis; UE3's depth fade is not ported (hard edge). Movable and tiled decals are not covered. Evidence: editor and in game.

## Post-process, fog and sky

- **Effective post-process settings:** UE3 `PostProcessVolume` override flags mostly default to TRUE, so flags the export omits still override. Derive struct defaults, then world settings, then each volume's overrides.
- **Local colour grading:** one bounded UE4 volume per source volume, with the source LUT, the exact hull, blend radius 0 and priority in source order. UE3 time fades have no UE4 match; the switch is instant (recorded).
- **Light shafts:** UE 4.25's light-shaft shader is the UE3 one, so shaft occlusion and bloom parameters map one to one (non-directional shaft lights recorded as partial). Copied 1:1 they were too strong in the Chivalry 2 scene; see [completion rules](completion-rules.md#atmosphere).
- **Bloom** (max-RGB threshold, clamped gather) has no faithful UE4 equivalent: base conversion records it, completion approximates it.
- **Height fog** derives from saved values plus the dominant light. Source fog also applies per vertex to translucent and additive materials and particles, which the opaque pass does not cover.
- **Custom lighting needs a light pass.** UE3 evaluates custom-lighting materials only for components that accept lights. A sky dome with `bAcceptsLights=False` never drew its material sun.

## Foliage and motion

Place foliage as hierarchical instances with source transforms, seed and per-instance data. Decide world-position-offset motion from rendered uses (mesh slot or override, per LOD), not from reference reach: meshes keep stale material references. Check the wind input only in graphs that read it. CMW bush sway stretched in UE4 and is a candidate for replacement during completion. Static recovery of cloth or skeletal props is not simulation.

## Effects

- Rewrite classes and properties without a UE4 equivalent only when exact; otherwise record them. Refuse and record systems using a class missing from UE 4.25 instead of letting the editor stage fail.
- UE3 SubUV with no interpolation shows the whole texture, but UE 4.25 sprites always sample one sub-image. Set sub-images to 1×1 for those modules (seen in game as tiled squares).
- UE3 mesh-emitter pre-rotation becomes a fixed `RollPitchYawRange` (min equals max).
- Apply owner scale only on world-space emitters. Local-space emitters already inherit the actor scale; forcing it scaled them twice (in game).
- Create particle LOD levels under their emitter, modules and distributions under the system. A component's `EmitterMaterials` is transient in UE 4.25 (per-emitter overrides are not saved). UE 4.25 Python does not expose Cascade modules; editing them needs native code.
- **Crash at load (in game):** an auto-activated placed emitter whose system has mesh emitters crashed Chivalry 2 about 50 s after map load. Probe maps ruled out mesh asset, loop count, kill-on-completed and emitter space; sprite-only systems loaded. Mechanism unknown. Workaround: one-shot mesh-emitter systems start deactivated. One looping mesh system loaded fine elsewhere, unexplained.

## Audio

- Convert cue graphs whose root is one attenuation node over loop, modulator, delay, random and wave nodes, keeping node order (a modulator outside a loop draws once per play). Index mixer weights by source slot. `AmbientSoundSimple` inline nodes become cue graphs; auto-activation follows the class's `bAutoPlay`.
- Map distance models by name; fail on an unknown one, never fall back to Linear. Oscillators, loop counts and continuous modulators stay unsupported. Cues without attenuation play non-spatialized at full volume in both engines.
- Infinite-plane sources become a world-aligned box: half-extent `radius_min` on the plane's distance axis, map bounds plus `radius_max` on the other two, falloff `radius_max − radius_min`.
- Reverb presets need an explicit adapter; a correct region shape does not establish matching DSP.

Volume sliders and Wwise: [completion rules](completion-rules.md#sound).
