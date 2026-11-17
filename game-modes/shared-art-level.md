# One art level for several game modes

How one map's art can be stored once and used by an FFA map and a Team Objective (TO) map in the same mod PAK, the way retail Chivalry 2 shares `Courtyard_Art` between its FFA, TDM, LTS and Arena maps. FFA setup is covered in [FFA map setup](ffa-map-setup.md) and the staged match in [Team Objective maps](team-objective-maps.md).

Evidence (UE 4.25.4 ArgonSDK, Unchained, 2026-11-12 to 2026-11-17). Engine rules were read in the UE 4.25 source and exercised in the editor. Packaging was checked by listing the cooked PAK. In the shipping game: a small probe map (listen server and one joined client), then a full converted map, whose FFA map and TO map both loaded from one PAK. Late join and dedicated servers were not tested separately. The TO map's first build showed a duplicated prop. Its cause is described under "Copies must match the original". The corrected build has not been tested in game yet.

## Layout

| Level | Holds |
| --- | --- |
| Art level | Everything that does not depend on the mode: geometry, lights including the sun, baked lighting (its own `_BuiltData`), fog, sounds, effects, mode-independent traps and damage volumes. |
| FFA map (`FFA_...`) | A thin persistent level that streams the art level: world settings with the FFA mode type and map summary, kill Z, navmesh and bounds, spawners, waves, match startup. |
| TO map (`TO_...`) | A thin persistent level that streams the same art level: world settings with the TO mode type and its own map summary, kill Z, navmesh, spawners, stages and objectives, and copies of the art actors the match moves (below). |

The gameplay maps add the art level as an **always loaded** streaming level, which loads with the map, so RAM use is the same as one combined map. The saving is disk space and download size. The art level and its build data are stored once, and the cooker pulls them in through either gameplay map.

The launcher lists maps whose world settings have a game mode type. Leave the art level's mode type at the default, so it is not listed. Both gameplay maps are listed from the same PAK.

## Engine rules that shape the design

1. **Build data is per level.** Each level has its own `<Level>_BuiltData` package. A component finds its light maps in its own level's data, whichever persistent level streams it. Bake or import lighting with the art level open as the persistent level: a lighting build with the art loaded under a gameplay map rebuilds that level's data from the wrong world and wipes the art level's.
2. **Moving actors between levels drops their light maps**, and so does duplicating a world. Create the split before writing any build data. Author the art directly into the art level; do not move it there afterwards.
3. **No hard references from one map into another map's actors.** Saving fails. Soft object references save. In game they resolved on the listen server and on a joined client.
4. **Gameplay maps hold no build data.** A Static or Stationary primitive placed in a gameplay map renders without its static lighting, so gameplay maps should contain only Movable or non-rendering actors. The gameplay maps can keep World Settings "Force No Precomputed Lighting" on. The art level's baked lighting still rendered under them in game.
5. **Saving.** `SaveMap` saves only the persistent level's build data. Saving the current level (`save_current_level`) saves that level and its own build data. A "save as" of a map saves only its persistent level, so a thin map can be copied this way to start a second mode.
6. **Streaming transforms cannot scale.** A streaming level's transform moves and rotates but never scales, so a scaled conversion scales each actor in the art level.
7. **Adding an existing level to another map** is a C++ call (`AddLevelToWorld`). Editor Python can only create a new streaming level.
8. **Names:** avoid level names ending in `_<number>` (the engine splits the number off the name).

## Changing art actors from a gameplay map

A TO match changes the art: it hides props when a stage ends, shows burnt remains, moves gates and carts, and plays effects on art actors. The gameplay map cannot hold hard references into the art level (rule 3), and it must not save the art level, because the other mode shares it. Two techniques cover every case:

- **Soft references, resolved at runtime.** Stage rows hold soft actor references to art actors to show or hide. A "hide at start" list hides art actors that the TO match needs gone from the beginning. Effects hold a soft target. On every machine a short timer resolves the paths (resolve only, no load: the art level is always loaded) and applies the current state. It retries targets that are not resolved yet, which covers joining clients. A start-hidden actor may show for one timer period (0.25 s in our build) at match start or on join.
- **Movable copies in the gameplay map.** Actors the match moves (movers, carts and everything riding them, effects attached to a mover) and start-hidden actors that an effect shows again are copied into the TO map as Movable actors. The match moves or shows the copy, while the original goes on the hide-at-start list. A copy avoids two independent timers racing on a joining client: one hiding the original, the other showing it. Copies of Movable originals light the same as the original.

A verify step should resolve every soft path in the editor after reloading the saved TO map. A renamed or deleted art actor breaks a soft reference silently at runtime.

### Copies must match the original

A copy must carry every saved property the original's renderer and game state use, not only mesh and transform. Our first build copied mesh, material slots, transform, collision profile, shadows and tags, and the comparison with the earlier single-level build found three gaps:

- **Saved hidden state.** A prop that was saved hidden in the art level (it moved with a falling tree but was never meant to show) was copied as visible. In game it appeared as a second, bare tree trunk beside the fallen tree. Copy the original's hidden flag, and keep its collision setting.
- **Custom Primitive Data** (`StaticMeshComponent.custom_primitive_data`), the per-component shader parameters that materials read: copies lost them, so they could render differently from the original.
- **Override materials.** Writing every material slot from `get_material` fills slots the original left empty with the mesh's default material. It renders the same, but it is not the original's saved state. Copy `override_materials` as it is.

How it was found: both cooked map PAKs were dumped to JSON with a CUE4Parse-based reader, and every actor and component was compared property by property. One side was the earlier build, with the art and gameplay in one map. The other was the split build, its art level and TO map taken together, with each copy compared to its original. Apart from the expected differences (soft references, attachment to movers, light-build GUIDs, sub-centimetre float noise), only those three gaps remained. Such a full comparison after a structural change found what a visual check missed.

## Packaging

Cook both gameplay maps (for example `-map=FFA_Map+TO_Map`). The cooker cooks the streamed art level and its `_BuiltData` because the gameplay maps reference them. One mod PAK then holds the art once and both gameplay maps. Check the PAK listing: the art umap, its `_BuiltData`, both gameplay maps and no stray editor-only packages.

## Not covered

- Distance-based streaming of art sublevels (a separate project; always loaded is simpler and matches the retail mode-sharing maps).
- Reflection captures: none were used. If added, capture with the art level open as the persistent level.
- A per-mode navmesh rebuild: the TO map here kept a navmesh built for the FFA layout, with dynamic generation updating tiles when hidden art turns its collision off.
