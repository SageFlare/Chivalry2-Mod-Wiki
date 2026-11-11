# Completion rules and Team Objective setup

Completion turns the faithful base into a playable Chivalry 2 FFA map with automatic rules. Everything base conversion recorded as "no direct conversion" lands here.

## How the rules run

- Rules run in a fixed order on a copy of the base map; the base stays untouched. Each rule builds on the verified result of the rule before it, verifies its own work in a fresh editor process and rolls itself back on failure. Rules can be undone newest-first, which lets the base be regenerated.
- Each rule derives its values from source data with one formula for every map. Per-map tuning is manual work after completion.
- Order used: atmosphere → scale → sound (Wwise bank) → slider-volume fallback (SoundMix) → damage volumes → siege/resupply replacements → traps and other map logic → FFA → light-map import ([light maps](light-maps.md)).
- Rules that add primitives run before the FFA and lighting rules, so navigation and build data cover the finished map.

## Atmosphere

- **Height fog:** UE3 and UE4 both divide saved density and falloff by 1000 before rendering. Copy the saved values; copying renderer-scale values made the fog 1000× too thin. Height, start distance and maximum opacity carry over. A single uniform colour, blended from the source's light and opposite colours, replaced UE3's split colouring.
- **Bloom:** an unbound bloom-only post-process volume above the base conversion's post-process volume. Threshold equals the source threshold. Intensity is the source scale times one global factor for Chivalry 2's scene brightness, calibrated to the official maps' atmosphere data (median bloom scale) and then checked in game. Two UE4 bloom stages carry the source tint (UE4 scales each stage by 1/6).
- **Light shafts** use the same global factor. Copied 1:1 they were too strong on every map.

Evidence: editor, package and in game on six maps. Unverified: bloom size mapping, threshold after exposure, translucent fog against CMW's per-vertex fog.

## Scale

CMW's player collision is 130 units tall; Chivalry 2's playable class defaults give 176, a ratio of about 1.354 (source: CMW SDK pawn defaults, Chivalry 2 SDK Blueprint defaults). Widths give about 1.25, so one scale cannot match every clearance. A uniform **1.35×** scale about the origin was the default (in game, several maps). Pick your own factor from traversal tests.

- Scale everything spatial: collision, terrain, attachments, volumes, effect and audio extents, the precomputed light volume.
- Materials that compute from absolute position (`WorldPosition`, `ObjectPositionWS`, `CameraPositionWS`, object radius) do not follow the scaled actors. Terrain paint landed in the wrong place until every such read got a ÷S divide (in game).
- Local-space particle emitters already inherit actor scale; see [effects](materials-and-environment.md#effects).

## Sound

Chivalry 2's volume sliders drive Wwise buses only, so UE4 sound cues ignore them. Converted cues went into a per-map Wwise bank, packed loose under `TBL/Content/WwiseAudio/Windows/` and loaded by name at runtime. A probe bank followed the sliders in game. Whether converted CMW sounds follow them was not yet confirmed. Sounds left on UE4 audio get a fallback rule: a SoundMix override polled every 0.5 s, which follows the sliders in game. Building a bank needs Audiokinetic's tool under its licence, or community format work; this wiki supplies neither.

## Chivalry 2 replacements

- **Burn damage:** the `FireDamage` damage source exists only in the retail game package; the SDK copy is an assembly stand-in. Reference it by path and resolve it at runtime. A volume without a resolved source disables its tick.
- **Siege weapons and resupply:** placed Chivalry 2 Blueprint instances cooked with the SDK stubs' components. Spawn the real classes at runtime from marker actors instead.
- **Facing:** take a replacement's facing from the CMW **drawn mesh frame**, not the actor yaw. UE3 applies a skeletal mesh's `RotOrigin` before the actor transform. Compare the long axes of both meshes before trusting a yaw.

Evidence: the siege replacements and their facing were confirmed in game; the burn damage rule was built, but its damage was not specifically confirmed in game.

## Map logic

- Classify each Kismet chain by its trigger. Use, touch, level start and timer chains belong to completion. Objective events, stage changes, spawn changes and match end belong to objective setup. A chain needing both is split.
- **Levers:** Chivalry 2's simple lever Blueprint is a wall lever (back face at local Y 0, arm out of +Y, swinging about X). On a floor it needs roll −90. Take its yaw from where the CMW lever's own movement track moves the tip, not the CMW actor yaw. Hide the CMW lever base meshes. Evidence: the wall-pose bug was seen in game; the floor pose fix was installed, in-game confirmation pending.
- State that only objectives change (objective gates, stage-revealed objects) keeps its CMW start state on an FFA map.

## FFA

General setup: [first map](../create-first-map.md), [spawns and navigation](../spawns-and-navigation.md), [FFA lifecycle](../ffa-lifecycle.md).

- One single-slot spawner per CMW start (the game's own start class and subclasses), with the source yaw. Plain `PlayerStart` actors are unused by CMW modes and stay unused.
- One navmesh per map. Its bounds are the union of the ported blocking volumes and the starts plus a margin, with runtime generation set to Dynamic on the instance. Record a start off the navmesh; never use it as the path root. Kill volumes become null navigation areas (implemented; untested on a map that has kill volumes).

Evidence: in game on several maps. Unverified: bot behaviour (saved-data checks only), spawners on reachable roofs.

## Objective setup: Team Objective (proposal)

**Not built.** Planned from a static audit of CMW source:

- Do nothing on a map without objective source data (every duel map).
- A stage ends when Kismet reaches the next objective-activation action (stage + 1, bonus time added) or the end-game objective action. Take order and stage ends from those nodes, not from stored objective lists, which were empty on several maps.
- Per-team spawner wiring per stage. Gates: collision off or a navigation-area switch when open. Bot routing through destination volumes. One navmesh built with every stage's geometry, as official maps do.

Manual work that no stage generates: extra spawns, cameras and cinematics, end-of-match podium, weapon racks and pickups (CMW maps have none), navigation links and cost areas.
