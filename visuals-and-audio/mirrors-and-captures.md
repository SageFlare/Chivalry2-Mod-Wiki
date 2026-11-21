# Mirrors and scene captures

These are bounded observations from UE 4.25.4 and Chivalry 2 shipping-game experiments. No mirror library or custom implementation is distributed here.

## Rendering choices

Planar reflections rendered black in the tested game while still costing about 2.4 ms. Screen-space reflections and reflection captures did not provide a reliable head-on mirror. SceneCapture-based mirrors rendered successfully; a prebaked cubemap sampled into emissive also worked as a glossy approximation. These results do not prove universal behavior across game versions or configurations.

A live planar mirror requires a reflected camera transform, correct projection and clipping. The tested game did not provide the expected global clip-plane support; an oblique near-plane projection was used experimentally. This is an implementation requirement to investigate, not a turnkey Blueprint recipe.

Avoid overlapping mirror faces and nearby backing geometry. Derive clearance from the actual panel construction; in one tested construction, geometry closer than about 4 cm behind the mirror face z-fought in the capture. Projection choices suitable for a large floor may produce incorrect wall reflections; test grazing angles and near objects rather than treating resolution as a universal fix. A screen-mapped projection suited large floor mirrors but duplicated or misplaced reflections on wall mirrors. Portal-style captures pixelated close reflections at grazing angles, and a higher capture resolution did not fix it.

## Visibility

First-person characters can hide third-person head and equipment meshes. Scene captures may have no view owner, so owner-visibility flags matter independently of ordinary hidden state. In particular, `bOnlyOwnerSee` can exclude sheathed items even after unhiding them. Test appearance in both the main view and capture before changing shared actor flags.

## Performance and lifecycle

- Measure resolution, cadence, staggering and capture count independently. Declare render-target dimensions explicitly; a quality scalar is not automatically a fraction of the player's screen.
- Profile both capture GPU time and per-frame Blueprint scheduling CPU time. Visibility/distance checks can cost more than the captures they avoid.
- Time-based scheduling avoids some frame-cap-dependent cadence changes. Measure responsiveness at multiple frame rates.
- Resolve disabled settings before allocating render targets or stand-ins. Test enabling later and settings that arrive after startup.
- Cache static transforms only when they are actually static; refresh after movement.
- Some show flags had no effect in shipping, and some applied only at component registration. Verify flags in the actual game rather than relying on editor previews. In the tested game, light-type and post-process show flags on a capture did nothing; disabling ambient occlusion in reflections saved about 1 ms. Flags that apply only at registration have to be set on the component template, not changed at runtime.
- Controls that did reduce cost in shipping: capture resolution, update cadence, staggering captures across frames, and capture distance. With many mirrors, per-tick Blueprint checks became the bottleneck rather than the captures. Frame-counted cadences look choppy at a 60 FPS cap.
- Resizing a render target with `ReleaseRenderTarget2D` forces a render flush. Resize in place, and when several meshes share one material instance, change its texture parameter rather than creating new instances.
- Hiding the capture component stops rendering but keeps its setup, which is useful for on/off cost comparisons.

## Settings and telemetry

If sending settings to clients, use an explicit key allowlist and serialize only the required values. Do not replicate a complete server options string: it can contain unrelated or private settings. Validate asset identifiers before loading; a wrong-type cast after loading does not bound loading cost. Authority on an arbitrary local actor is not alone proof that it is the server.

Map URL options use `?Key=Value`; process arguments use their own syntax (`-Key=Value`). They are different input channels. Up to Unchained-Mods 0.2.2, the main-menu map **Options** box was the map URL (offline, you are the server), so it took `?Key=Value`; a `-Key` entry there was silently ignored, and launch arguments belong in the launcher's launch-argument field. Unchained-Mods 0.2.3 (November 2026) removed that box and the typed map name field, according to its release notes; we have not found another menu route for offline `?Key=Value` options. Launch arguments still work for process options. Editor `ke` (KismetEvent) calls are compiled out of shipping builds in UE 4.25 (`#if !UE_BUILD_SHIPPING`): a `ke` command did nothing in the shipping game, so use it only for editor probes.

Changing a Blueprint class default changes every placed instance that never overrode that value (UE stores only differences), including maps already shipped. Pin explicit values on maps that must not change.

Breakable game windows kept their swapped (broken) mesh visible in the tested game. A mirror tile built on them had to poll the breakable state (`IsDead()` or the breakable state reaching broken) and hide itself.

For performance probes, sample real frames, record the test settings, and verify completion of SaveGame writes before leaving. In the shipping game, `SaveGameToSlot` wrote to `%LOCALAPPDATA%/Chivalry 2/Saved_Unchained/SaveGames/` when launched through Unchained. Use fixed, simple slot names: UE 4.25 builds `<Saved>/SaveGames/<Slot>.sav` without sanitizing the slot name (source inspection), and the same folder holds the mod loader's own `ModLoader` slot, which must never be reused. See [Reviewing third-party PAKs](../game-research/pak-review.md) for the security side. Capture screenshots and editor timings do not establish shipping appearance or multiplayer behavior.
