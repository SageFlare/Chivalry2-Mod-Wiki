# FFA map setup

How to make a custom FFA map play like the official FFA maps: world settings, kill cap, start timing, bot roaming and cinematics. The values come from official maps and the game's packed data, inspected read-only; the procedure was then applied to a custom FFA map and checked in the shipping game (2026-10-25, UE 4.25.4 ArgonSDK). The respawn recipe and navigation checks are in [Spawns and navigation](spawns-and-navigation.md); the object model and open questions are in [FFA lifecycle](ffa-lifecycle.md).

The game's own Blueprints run at play time; SDK stubs only provide names. Check an official map and the packed config before guessing a value.

## World settings

| Field | Official FFA maps | Notes |
| --- | --- | --- |
| `default_game_mode` | `Mode_FFA` | |
| `attacking_faction` | FFA (one map uses All) | |
| `game_mode_settings` | an instanced `Settings_FFA` object owned by the world settings | holds Max Score and match time; 4.25 Python has no `new_object`, so create the instance in the editor or with a native helper |
| `map_summary` | an instanced `TBLMapSummary` owned by the map package | name, description, faction, mode type |
| initial spawn | the big GroupSpawner in `defender_initial_spawn`, attacker slot empty | the attacker slot also worked |
| `game_mode_message_table` | `MessageTable_FFA` | match-start banner and kill callouts |
| prologue / waiting / epilogue sequences | LevelSequenceActors in the map | see Cinematics |

**Message table:** the table is not in ArgonSDK. Create an empty DataTable (row struct `GameModeMessageTable`) at the game's own path, `/Game/GameModes/MessageTables/MessageTable_FFA`, outside your mod folder, and reference it. Only the mod folder is packed, so at runtime the reference resolves to the game's real table. Verified in game: the "FREE FOR ALL" banner and kill callouts showed. Never ship the placeholder.

Blueprint variables with spaces are reached from Python by their raw names, not display names (see [editor automation](../editor-and-scripting/editor-automation.md)).

## Spawning

- Official maps spread many one-slot `BaseSpawner`s (one map has 63). Multi-slot spawners also work: **columns run along the actor's forward axis, rows along its right axis**. The construction script rebuilds the slot list from the actor, so place and size the actor and let the script rerun; do not write the cached slot data by hand.
- Each spawner and the group use the FFA spawner score method and a spawning minimum distance of 800 units, with only the FFA faction override in the wave parameters.
- **The SpawnSequence should hold a single entry: the GroupSpawner** containing all spawners. Listing the spawners individually made each wave fill one spawner, giving very slow respawns. Official timing: 6 s between waves, one simultaneous wave.
- Capacity: players and bots beyond the slot count stay dead. `Mode_FFA` uses a 5 s minimum respawn delay.

## Win condition (kill cap)

Place a `ScoredStage` actor (`/Game/GameModes/Stages/ScoredStage/ScoredStage`) and give it the same Max Score as the `Settings_FFA` instance. The stage brings the objective bar, time objective and cinematic components; no level-script wiring was needed.

`Settings_FFA` defaults are Max Score 35 and match time 900 s. Bonus curves add nothing up to 24 players, then rise linearly to +40 at 64 players, and bots count. Verified in game: a base of 40 showed 80 with 80 bots, the 900 s timer applied (not the stage's own shorter timer), nearing-victory banners and the end screen appeared, and the match ended at the cap.

## Match start timing

The start waits (`MinTimeBeforeStartingMatch`, `MaxTimeBeforeStartingMatch`) and warmup time are config values. The packed game config sets a 45 s maximum and a 0.75 desired-players fraction and leaves the minimum at its native default. An observed default start took about 30 s.

**Do not change these from map logic at runtime.** They are config properties and can end up saved in the player's local game config, then apply to every map. Zeroed values were once found there; whether a runtime change or an earlier manual edit caused it was not established. For quick local starts, edit your own config deliberately and revert it. Servers set these through server config or the launcher's warmup option. A map cannot change the start wait or prologue length per map.

## Bot roaming

Default bots roam only toward destination points tagged `AI.DestinationVolume` (a low-weight "walk around objectives" behaviour, so combat still wins). Without such points, bots idle until they meet an enemy.

- Place one `DestinationGeneratorVolume` over the playable area with `AITagsToAddWhenActivated` = `AI.DestinationVolume`, as an official FFA map does. Activation was left at its default.
- The SDK's `GeneratePoints` is a stub, so compute points yourself: grid points projected onto the navmesh, stored in `DestinationListRelative` as `(world - volume location) / volume scale`. Official spacing is 8 m; 4 m suited narrow corridors.
- Observed: bots pick points roughly 10-30 m away and keep no memory, so long dead-end corridors can make them walk back and forth.

## Cinematics

- In-game order: prologue (on joining) → waiting-to-start (plays the waiting sequence until the match starts, cutting it mid-shot) → preparing match → playing. The epilogue is the podium and needs a podium setup, which was not tested.
- The game holds the prologue for 10 s plus a fade. A shorter sequence repeats; official prologues are 15 s and get cut, so make yours 15 s.
- The waiting sequence can be a master LevelSequence with a cinematic shot track (one sub-sequence per camera shot) and a fade track. With a ~30 s start, about 20 s remains for it.
- Authoring sequences from Python needs the engine's SequencerScripting plugin enabled for that editor run.
- Official maps play a map-name voice line from the prologue via a Wwise event track. A plain SoundWave on an audio track in your prologue was audible in game. Raise its Volume above 1 for loudness while keeping the peak below 0 dBFS. Such sounds are UE4 audio, so they very likely ignore the game's volume sliders (inferred, not tested for sequence tracks); see [Mod audio](../visuals-and-audio/mod-audio.md).
- Choose camera paths along open navmesh corridors, away from map-edge walls.

## Not possible from a map

- The loading-screen image: the game's loading-screen config lists only official maps, so custom maps get the default image.
- Prologue length or start wait per map (game Blueprint and config).

Other modes follow the same pattern: a `Settings_<Mode>` instance, the mode's stage class and the same cinematic states. Check that mode's official maps first. Staged objective matches are covered in [Team Objective maps](team-objective-maps.md).
