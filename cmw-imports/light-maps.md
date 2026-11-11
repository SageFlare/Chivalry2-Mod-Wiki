# Light maps

CMW maps ship precomputed lighting. Converting it avoids a new bake and keeps the original look. Three routes were used:

| Route | Pass | Look | Notes |
| --- | --- | --- | --- |
| Unlit adapter materials decode CMW light maps | base conversion (faithful reference) | Closest to CMW | Every CMW light must stay Movable, so dynamic shadow cost is high |
| CMW light and shadow maps written as UE4 build data | completion (default) | Close; somewhat darker on one map, accepted | Needs native editor code; no Lightmass |
| New Lightmass bake | completion (trial) | UE4's own; needed calibration | Bake memory grows with the map |

## Source data (source)

- **LightMap2D:** per component and LOD, two directional coefficient textures, scale vectors and an atlas scale and bias. The texel decode is tangent-space with a fixed three-direction basis (`Diffuse · a · Σ bᵢ · saturate(N·Bᵢ)²`).
- **ShadowMap2D:** a distance-field shadow per dominant light, decoded `saturate((d + x) · y)^z` with z = 2. UModel 1590 does not register this texture subclass. Exporting a disposable copy with a class alias decoded the inherited pixels, checked against a GPU capture of the original. Never patch the installed packages.
- **LightMap1D (vertex lighting):** two BGRA colours per vertex. CMW's vertex shader decodes `pow(byte/255, 2.2) · scale` **before** interpolation, which differs from interpolating encoded bytes.
- **Dominant directional light:** a static profile. Direction = −`DominantLightShadowInfo.LightToWorld` Z plane; colour = (LightColor/255)^2.2 × Brightness; signed-distance parameters from `LightSourceAngle` and `ShadowExponent` (default 2). This reproduced captured constants on two maps, so new maps need no capture.
- A mesh export without `LightMapCoordinateIndex` uses UV0. Shadow maps exist per LOD and can live in their own texture. Light- and shadow-map texture names repeat across level packages, so key them by level.
- A component with empty or type-None light-map data was lit dynamically in CMW. Keep it on movable lights.

## Route 1: adapter materials

Each lit material gets an Unlit duplicate (the adapter) that decodes the CMW coefficients and shadow factor and multiplies by the material's diffuse.

- **Lighting models:** non-directional = constant transmission 1; one-sided masked = the opaque formula with its mask; soft-masked and translucent lighting stay refused. The adapter's blend mode follows the imported material, not the export's `BlendMode`.
- **Vertex-lit components:** a component-specific mesh copy carries the six decoded coefficients in three extra full-precision UV channels. The pixel shader applies the directional basis. A vertex light map on LOD k under a texture-lit LOD0 gets a per-component copy with coefficients on that LOD only.
- **BSP and terrain** use the same route, bound per surface group. Accept a degenerate shadow coordinate only on slivers under 1 cm².
- **Packing:** repack light and shadow maps losslessly into power-of-two atlas pages (at most 4096, edge-replicated padding). Store each component's atlas transform in **custom primitive data**: write the saved property, because the `SetCustomPrimitiveData*` setters are transient. Use this only when every sampled UV is proven inside [0,1]; otherwise use a per-slot instance with clamping. Without a shadow map, keep the shadow factor 1, never 0. This cut material instances by about an order of magnitude.

Evidence: editor, package and in game on several maps. Limits: Unlit surfaces do not respond to dynamic lights, the movable lights cost frame time, and map load stayed slow.

## Route 2: import as UE4 build data

UE3 and UE 4.25 store equivalent data. Write it the way the engine does after a Lightmass import:

1. `InvalidateStaticLighting` for the level's build-data id only, so an existing precomputed light volume survives.
2. Per component and LOD: create build-data ids, `AllocateMeshBuildData`, `FLightMap2D::AllocateLightMap`, `FShadowMap2D::AllocateShadowMap`. Foliage uses the instanced variants with per-instance biases.
3. `EncodeTextures`, then `SetupLightmapResourceClusters` and `InitializeRenderingResources`.

Rules and failures:

- Allocating into a registry whose resource clusters are already set up hits `check(!bSetupResourceClusters)` (editor crash). Invalidate first.
- Give every LOD an id (LOD data count = mesh LOD count). Partial LOD data made lookups fail for all LODs.
- **Every Static light needs a light build-data entry.** Otherwise it renders dynamically on top of the light maps.
- Set the sun's dynamic cascaded-shadow distance to 0 so the imported shadow maps provide its shadows.
- Movers placed Movable hold no light map: skip and record them (dynamic lighting plus indirect lighting cache). Components without CMW data become Movable with a single cache sample.
- **Per texel:** keep CMW's luminance at the geometric normal exactly. Fit UE4's world-space L1 directionality by least squares; that fit is the only approximation in the light map. Shadow distance fields remap linearly per light.
- Some foliage instance biases point past their texture. Read them with UE3 wrap addressing; it reproduces CMW's own (wrong) lighting.
- **Verify** by decoding sample texels from the encoded textures in a fresh editor process. Worst error was 1.3 %.

Evidence: editor, package and in game on four maps. Limits: L1 fit on normal-mapped pixels, no light-map specular, no static shadow depth map for translucency, brightness factor not calibrated, frame rate not measured. Engine source references: `MapBuildDataRegistry.h`, `LightMap.h`, `ShadowMap.h`, `LightmapCommon.ush`.

## Route 3: Lightmass

- A native bake needs CMW's light-map UV channel per mesh. Many imported assets carried index 0 where CMW used 1; re-index on map-local duplicates. Vertex-lit props have no light-map UV and stay Movable.
- UE 4.25 gives each foliage instance its own mapping, so foliage-heavy maps multiply bake memory. `bUseEmbreeInstancing` (Lightmass `DevOptions.StaticLighting`) cut ray-tracer memory without changing the lighting. Lightmass thread count comes from the Swarm agent's `LocalJobsDefaultProcessorCount`, not editor lighting options.

Evidence: editor, package and in game on one map; superseded as default by route 2.

## Character lighting

CMW's precomputed light volume was written as UE4 precomputed volume lighting (indirect lighting cache) in the last authoring step of base conversion. Later lighting writes must preserve it. Evidence: in game.
