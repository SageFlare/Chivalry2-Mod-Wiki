# Mirrors and scene captures

These are bounded observations from UE 4.25.4 and Chivalry 2 shipping-game experiments. No mirror library or custom implementation is distributed here.

## Rendering choices

Planar reflections rendered black in the tested game while retaining measurable cost. Screen-space reflections and reflection captures did not provide a reliable head-on mirror. SceneCapture-based mirrors rendered successfully; a prebaked cubemap sampled into emissive also worked as a glossy approximation. These results do not prove universal behavior across game versions or configurations.

A live planar mirror requires a reflected camera transform, correct projection and clipping. The tested game did not provide the expected global clip-plane support; an oblique near-plane projection was used experimentally. This is an implementation requirement to investigate, not a turnkey Blueprint recipe.

Avoid overlapping mirror faces and nearby backing geometry. Derive clearance from the actual panel construction. Projection choices suitable for a large floor may produce incorrect wall reflections; test grazing angles and near objects rather than treating resolution as a universal fix.

## Visibility

First-person characters can hide third-person head and equipment meshes. Scene captures may have no view owner, so owner-visibility flags matter independently of ordinary hidden state. In particular, `bOnlyOwnerSee` can exclude sheathed items even after unhiding them. Test appearance in both the main view and capture before changing shared actor flags.

## Performance and lifecycle

- Measure resolution, cadence, staggering and capture count independently. Declare render-target dimensions explicitly; a quality scalar is not automatically a fraction of the player's screen.
- Profile both capture GPU time and per-frame Blueprint scheduling CPU time. Visibility/distance checks can cost more than the captures they avoid.
- Time-based scheduling avoids some frame-cap-dependent cadence changes. Measure responsiveness at multiple frame rates.
- Resolve disabled settings before allocating render targets or stand-ins. Test enabling later and settings that arrive after startup.
- Cache static transforms only when they are actually static; refresh after movement.
- Some show flags had no effect in shipping, and some applied only at component registration. Verify flags in the actual game rather than relying on editor previews.

## Settings and telemetry

If sending settings to clients, use an explicit key allowlist and serialize only the required values. Do not replicate a complete server options string: it can contain unrelated or private settings. Validate asset identifiers before loading; a wrong-type cast after loading does not bound loading cost. Authority on an arbitrary local actor is not alone proof that it is the server.

Map URL options use `?Key=Value`; process arguments use their own syntax. They are different input channels. Editor `ke` calls are not a shipping-game control interface.

For performance probes, sample real frames, record the test settings, and verify completion of SaveGame writes before leaving. Capture screenshots and editor timings do not establish shipping appearance or multiplayer behavior.
