# Lighting and materials

These UE 4.25.4 findings include editor/package validation and limited shipping-game observations. They are not a complete survey of light types.

## Baked lighting

1. Set `force_no_precomputed_lighting=False` and use suitable static geometry and lights.
2. Save every material used by a static mesh with **Used with Static Lighting** (`bUsedWithStaticLighting`), including Unlit materials. Instances inherit this from their parent. Missing saved flags caused grey default materials in game even when the editor appeared correct.
3. In Python, set `used_with_static_lighting`, recompile and save the material before baking or cooking. Validate the saved flag **before loading the map**, because map loading can set it only in memory.
4. Supply a valid lightmap UV channel. Choose resolution based on surface area and quality: roughly 5–8 cm per texel for detailed walls and 12–24 cm for large surfaces were useful experimental starting points, not universal requirements.
5. Avoid coplanar overlapping faces; baked surfaces exposed visible flickering.
6. Bake with the matching engine's commandlet, for example:

```text
UE4Editor-Cmd.exe <absolute-project> -run=resavepackages -buildlighting -quality=Production -allowcommandletrendering -map=<MapName>
```

Check exit status, bake logs and freshly saved `<Map>_BuiltData.uasset`. An `Illumination:` marker is useful evidence, not sufficient proof by itself. Treat `needed to have new flag set bUsedWithStaticLighting` for an owned material as a failed validation gate. Include BuiltData and any bulk companions in the cooked archive.

Static lights contribute through the bake. Movable geometry does not become a baked light emitter; character lighting needs separate runtime validation.

## Dynamic and emissive lighting

For an intentionally dynamic map, use Movable lights and set `force_no_precomputed_lighting=True`. Measure shadow cost in game.

Unlit emissive makes a surface bright. For baked spill, enable the static component's Lightmass `use_emissive_for_static_lighting`; a nearby static colored point light is another deliberate artistic choice. Bloom strength and actual illumination are different effects. The component exposes `lightmass_settings`; not every world/material Lightmass property is available through 4.25 Python.

Stationary lights and all skylight/directional combinations were not established by the source experiments. Test them separately.

## Python material quirks

- `TextureSampleParameter2D` needs a real default texture, such as `/Engine/EngineResources/DefaultTexture`.
- Abs and OneMinus connections used an empty input name.
- Some component-mask flags were not reliably configurable through Python; verify the saved graph or construct equivalent arithmetic.
- Read parameter values back after setting them. Cubemap parameters may require editing `texture_parameter_values` directly.
- Allow end-of-frame updates before judging dynamic material previews.

String presence in a cooked asset is a diagnostic clue, not proof of property value or correct rendering.
