# Lighting and materials

These UE 4.25.4 findings include editor/package validation and limited shipping-game observations. They are not a complete survey of light types.

## Baked lighting

1. Set `force_no_precomputed_lighting=False` and use suitable static geometry and lights.
2. Save every material used by a static mesh with **Used with Static Lighting** (`bUsedWithStaticLighting`), including Unlit materials. Instances inherit this from their parent. Missing saved flags caused grey default materials in game even when the editor appeared correct.
3. In Python, set `used_with_static_lighting`, recompile and save the material before baking or cooking. Validate the saved flag **before loading the map**, because map loading can set it only in memory.
4. Supply a valid lightmap UV channel (duplicated engine basic shapes have one); a mesh without one must stay Movable. Override lightmap resolution per component (`override_light_map_res`, `overridden_light_map_res`) based on surface area and quality: roughly 5–8 cm per texel for detailed walls, 12–24 cm for large floors and ceilings, and about 32 px for thin strips were useful experimental starting points, not universal requirements. Coarser values look blockier.
5. Use Static lights with `cast_shadows` enabled for baked shadows.
6. Avoid coplanar overlapping faces (for example wall cores running into pillars); baked surfaces exposed visible flickering. Trim geometry so faces do not overlap.
7. Bake with the matching engine's commandlet, for example:

```text
UE4Editor-Cmd.exe <absolute-project> -run=resavepackages -buildlighting -quality=Production -allowcommandletrendering -map=<MapName>
```

Check exit status, bake logs and freshly saved `<Map>_BuiltData.uasset`. An `Illumination:` marker is useful evidence, not sufficient proof by itself. Treat `needed to have new flag set bUsedWithStaticLighting` for an owned material as a failed validation gate. Include BuiltData and any bulk companions in the cooked archive.

Static lights contribute through the bake and affect only Static meshes that way. Movable meshes and characters receive baked indirect light only from the volumetric lighting samples. Movable geometry does not become a baked light emitter; character lighting needs separate runtime validation.

## Dynamic and emissive lighting

For an intentionally dynamic map, use Movable lights and set `force_no_precomputed_lighting=True`. Measure shadow cost in game.

Unlit emissive makes a surface bright. Variants built on a test map and loaded in the shipping game (the map worked; no preference between them was recorded):

| Want | Setup |
| --- | --- |
| Strip glows, nothing lit around it | Static or Movable Unlit emissive material |
| Glow tints nearby baked surfaces | Static mesh, component Lightmass `use_emissive_for_static_lighting` on, `emissive_boost` about 1 |
| Sharper coloured spill | Static mesh plus a small Static coloured point light in front (`source_length` = strip length) |
| Stronger bloom halo | higher emissive multiplier |

Bloom strength and actual illumination are different effects. The component exposes `lightmass_settings`; not every world/material Lightmass property is available through 4.25 Python.

Stationary lights and all skylight/directional combinations were not established by the source experiments. Test them separately.

## Python material quirks

- `TextureSampleParameter2D` needs a real default texture, such as `/Engine/EngineResources/DefaultTexture`.
- Abs and OneMinus connections used an empty input name.
- Some component-mask flags were not reliably configurable through Python; verify the saved graph or construct equivalent arithmetic.
- Read parameter values back after setting them: `set_material_instance_*_parameter_value` returned False even when the value was set, so check with the matching getter. Cubemap parameters may require editing `texture_parameter_values` directly.
- Allow end-of-frame updates before judging dynamic material previews.

String presence in a cooked asset is a diagnostic clue, not proof of property value or correct rendering.
