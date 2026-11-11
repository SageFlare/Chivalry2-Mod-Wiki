# Materials, lighting and environmental behavior

## Material conversion

Collect complete material graphs, instance inheritance, static switches, masks, texture identities, root settings and component slot overrides. A texture export is not a shader conversion. Preserve missing assignments as structured warnings instead of inventing textures or silently using defaults.

Validate saved graphs and mesh inputs independently, including any baked-lighting material variants. Existing graph state can survive repeat authoring; clear incompatible prior connections/settings or reject the transition. Tags describing an intended graph do not establish the saved graph's behavior.

Unlit water may ignore an attached normal input when the shader path does not consume it. Feed the normalized source normal explicitly into source-derived reflection/Fresnel consumers in the proper space. Do not invent refraction merely to make a normal input active. Keep distortion and other unsupported channels explicit.

## Source baked lighting

Preserve lightmap/shadowmap textures, component bindings, atlas transforms, decode coefficients, UV channels and light associations. Source shaders and attributed GPU draws can clarify semantics; class names and texture appearance are insufficient.

Vertex-lighting decode can occur before interpolation, which is not equivalent to decoding an interpolated encoded value. Two-sided shading, alpha, shadow terms, color encoding and lower mips require separate checks.

Displaying recovered baked diffuse through an Unlit material can preserve part of the source appearance while losing specular and dynamic-light interaction. State those limits. It is not equivalent to generating native UE4 Lightmass data or full renderer parity.

For source terrain normals, inspect the actual packed channels and reconstruction rule. Verify generated section geometry and UV precision independently.

## Foliage and motion

Preserve instance transforms and per-instance attributes. World-position-offset fidelity depends on effective runtime inputs, not just the material graph. Identify sampled GPU draws by mesh, world position, pass and LOD before generalizing captured constants.

In the inspected UE3 vertex factory, instanced and ordinary meshes used different color semantics, and instancing replaced alpha with cull fade. Observed stride-zero white bindings on selected draws did **not** establish a universal white-color rule. Keep unresolved native buffer selection explicit; do not erase valid source colors to match one capture.

Static recovery of cloth or skeletal props does not establish continuous simulation. A target cloth solver, invented wind or altered gravity is a separately documented approximation.

## Effects, audio and regions

- Preserve particle module enabled states, emitter distributions, ownership and effective activation. Exported disabled modules should remain disabled.
- Scope hidden flags to the correct component: hidden editor sprites do not prove a particle emitter is inactive.
- Particle deactivation may stop spawning while existing particles fade. Do not require immediate destruction as the only success criterion.
- Normalize reversed bounding-box endpoints only when source/target engine behavior justifies it, retaining the original evidence.
- Preserve audio cue topology, delays, looping, attenuation, randomization and state triggers. Similar node names do not guarantee equivalent behavior.
- Treat reverb preset mapping, wet sends and filter conversion as explicit adapters. Correct region shape and enum names do not establish DSP or mix parity.
- Compare all corners/planes of source volumes, priorities and inherited flags. Do not enlarge a volume merely to pass containment checks.
- Preserve effective draw distances and source culling state. Runtime reculling requires its own verified semantics; do not apply source regions indiscriminately to synthetic target helpers.
- Fog, gamma, LUTs and output transforms can dominate apparent brightness. Use matched views and effective source state; uncontrolled exposure sweeps do not prove numerical calibration.

Environment conversion may require manual editor work where standard Python cannot author the relevant graph or property. This wiki supplies no custom native helper. Record remaining work rather than claiming automation through an unavailable API.

Finish with saved-asset checks, representative renders, package validation and shipping-game visual/listening/traversal tests. Each answers a different question.
