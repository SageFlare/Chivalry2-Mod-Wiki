# Create a first FFA map

This introductory sequence is based on documentation review, not an independently completed end-to-end tutorial. Use the [Field Guide](https://knutschbert.github.io/ArgonSDK-FieldGuide/) for editor screenshots.

1. In a working UE 4.25.4 ArgonSDK project, create `/Game/Mods/AgMods/ExampleMap/FFA_ExampleMap`.
2. Add a simple floor and walls with collision. Use owned basic meshes/materials and leave room for characters at every spawn. Begin with one persistent level.
3. Add lighting. Choose dynamic or baked deliberately using [lighting and materials](../visuals-and-audio/lighting-and-materials.md). If using a separate atmosphere sublevel, make it Always Loaded and include it in cooking.
4. Set World Settings: `attacking_faction=FFA`, `gamemode_type=FREE_FOR_ALL`, and the generated class of `/Game/GameModes/Modes/Mode_FFA`. Set an appropriate Kill Z, map name and description.
5. Place the SDK's `BaseSpawner` (`/Game/GameModes/Spawners/BaseSpawner`) in the full editor. Set rows, columns and spacing for the required player capacity (the Field Guide example uses 4 x 4 with 500 spacing); inspect every slot. Assign it to Attacker Initial Spawn. A generic Player Start is not a substitute for this setup.
6. For bots, add a `NavMeshBoundsVolume`, build navigation and inspect actual coverage (press **P** in the viewport). Test human spawning before adding bots, and keep the population within spawn capacity (with 16 slots, one human and at most 15 bots).
7. Save, reopen, [cook and package](../packaging-and-hosting/cook-and-package.md) the explicit persistent map and its dependencies. In the project's list of maps to package, use the package path (`/Game/Mods/AgMods/ExampleMap/FFA_ExampleMap`), without `.umap`, a `World'...'` wrapper or the `.AssetName` suffix that Copy Reference adds. Add an atmosphere sublevel too if you made one. Keep custom dependencies inside the map's folder so a folder-based packer includes them.
8. Load through Unchained using your release's map controls. Verify initial spawn, collision and movement in game.

Initial spawning is only a prototype. Add [respawn waves and match-state activation](../game-modes/spawns-and-navigation.md) for a repeatable match loop. Validate death, respawn, spectating and multiplayer separately.

Map loading and Blueprint-mod loading are different workflows. A map does not require a separate mod marker merely to exist; separately loaded mod actors have their own discovery and activation requirements.
