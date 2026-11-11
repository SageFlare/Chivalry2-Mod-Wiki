# Geometry, attributes and collision

## Transforms and identity

Compose actor, component and instance transforms, including negative/mirrored scale. Inspect imported bounds, axes, winding and normals against source evidence. A historical OBJ route required Y negation and winding reversal; this is adapter-specific, not a universal Unreal conversion rule.

Use full source identities and deliberate case normalization for lookup. Preserve distinct objects sharing a short name. Reconstruct terrain from dimensions, spacing, height data and section offsets rather than assumptions from one map.

## Preserve mesh inputs

OBJ loses channels needed by many shaders. Prefer a route that preserves topology, material sections, UV sets, colors, normals and tangents together. Check LODs separately.

In UE 4.25.4, the basic glTF importer displayed an options dialog even for an automated import task. The bundled Datasmith glTF route supported an assets-only import without that dialog. Disable unrelated material/light/camera/animation import and generated lightmap UVs when preserving source data. Verify your installed plugin interfaces before scripting them.

The tested Datasmith path converted linear vertex-instance RGB into sRGB-encoded stored bytes. For source byte masks, preprocessing a separate working glTF copy with inverse sRGB on RGB, leaving alpha unchanged, recovered expected bytes within quantization tolerance. Apply this only after proving the encoding path for the actual accessor format; it is not a universal color correction.

Exported nonwhite colors are not proof of real source colors. Native buffers revealed both missing and spurious exporter colors. Compare native indices, positions, color ordering and absent-stream evidence. Keep component paint separate from base mesh colors; cached painted-vertex data may be stale relative to current component color data.

## Recovering lost channels

If retrofitting a lost UV channel, establish an oriented one-to-one triangle correspondence using positions, existing UVs, winding and material sections. Opposite-facing overlapping triangles can share positions/UV0 yet have different lightmap UVs. Independent corner lookup can overwrite matches on narrow triangles.

Use cyclic corner permutations and reject unresolved ambiguity. Restore a missing source triangle only when its complete corner attributes and section identity are proven. A verified near-degenerate repair does not justify inventing arbitrary missing geometry.

Select tolerances from actual export precision. Consider full-precision UV storage and degenerate-removal settings when preservation requires them. UE4 can reconstruct bitangents from normalized cross products and handedness; compare the effective representation rather than an invalid raw assumption.

Reopen saved meshes and independently compare positions, triangle mapping, UVs, colors, normals/tangents, sections and bounds. A successful save/import is not an equivalence check.

## Collision

Collect mesh BodySetup/AggGeom, level blocking volumes, component collision overrides and effective query policies separately. Render triangles are not equivalent to authored simple hulls. A visually present mesh may intentionally be nonblocking because of inherited actor defaults.

Restore supported original hull shapes and component intent. Convex-element counts are distinct from primitive simple-shape counts in the tested editor APIs. Bounds plus enabled flags do not prove hull preservation.

Record unsupported shapes explicitly. A flat-ended cylinder cannot be represented exactly by the usual capsule primitive; a polygonal hull is an approximation requiring a declared replacement rule. Complex-as-simple collision and automatic convex generation are choices, not source parity.

Verify saved hull vertices/policies, representative surface rays, player-sized sweeps and intended routes. Preserve nonblocking placements and deliberate barriers. If changing collision on a shared mesh should affect only selected actors, duplicate the mesh first.

Scene scaling must include collision, terrain, attachments, effect/audio extents and other spatial data. Validate with float-precision-aware comparisons and actual player clearance; a scale factor is not established merely by comparing character height.
