# Team Objective maps

How a mod map can run a staged Team Objective (TO) match: a chain of stages, objectives that complete them, spawns that move forward per stage, and a winner. FFA setup is covered in [FFA map setup](ffa-map-setup.md); spawner basics in [Spawns and navigation](spawns-and-navigation.md).

Evidence: in the shipping game, one player, local Unchained match, 2026-11-10/11 (UE 4.25.4 ArgonSDK). A two-stage test map ran end to end, and a second version of the same map exercised the objective types listed below. Several players, bots and dedicated servers were not tested; see the last section.

## World settings

- `default_game_mode` = `/Game/GameModes/Modes/StagedGameMode`, `gamemode_type` = Team Objective.
- `attacking_faction` and `defending_faction` set (the test used Agatha attacking, Mason defending), `use_new_stage_id_numbers` enabled.
- `attacker_initial_spawn` and `defender_initial_spawn`: one GroupSpawner per faction over one-slot `BaseSpawner`s, each with a faction override in its spawn wave parameters. The spawner rules are the same as for FFA.
- `map_summary`: a `TBLMapSummary` instanced in the map package with name, description, attacking faction and Team Objective mode type. The end screen showed the mode and map name; whether it came from the summary or the world settings was not separated.
- One player was enough: the match left warmup by itself. No custom startup actor is needed, because the staged game mode starts the first stage.

## Stage chain

1. Place one `ScoredStage` actor per stage (`/Game/GameModes/Stages/ScoredStage/ScoredStage`) with a distinct `StageIdNum` (for example 10, 20).
2. Victory stages are your own Actor Blueprints implementing `StageInterface`. In `StartStage`: check authority, get the TBL game mode, call `AwardVictory` with the attacking or defending faction (an instance-editable bool can choose which). Place one per possible winner. Retail grant-victory stages were not needed.
3. Build the chain backwards, from the last stage, with the `Initialize ScoredStage` nodes (Next Stage, Timeout Stage, returning `Stage`). The last stage's Next is the attacker victory and its Timeout the defender victory. Each earlier stage's Next is the following stage's `Stage` output; its Timeout is the defender victory.
4. The level Blueprint implements `StagedLevelBlueprintInterface`. Its `GetFirstStage` function returns the first stage, and the retail staged game mode calls it. The test kept the chain-building logic in a placed director actor and let `GetFirstStage` find it, build the chain and return the first stage.
5. A stage completes when server-side `ScoredStage.AddScore` reaches its Max Score. The simplest test objective was a trigger armed on its stage's `On Started`: an attacking character overlapping it added enough score to finish the stage.

The initialize variants are **pure** nodes (no exec pins): the chain is built when the graph reads their output. The "No Settings" variant forces Max Score 100 and 600 seconds. The variant taking a `ScoredStageProperties` struct (`InitialScore`, `MaxScore`, `DurationSeconds`, `PercentTimeToShareWithNextStage`, `MaxTimeToShare`) was verified in game; the time share appeared on the match clock. A stage that must keep its own length needs the previous stage's share set to 0.

Observed in game: the objective bar follows the current stage, time is added on completion with a sound, the final stage shows the attackers' victory, and the end screen lists per-stage times.

## Spawns per stage

- First stage: one placed `SpawnSequence` per faction, holding that faction's GroupSpawner, activated on the server from the first stage's `On Started`.
- Later stages, as retail maps do it: on the stage's `On Started` (server), `Deactivate` the old attacker sequence, spawn a new `SpawnSequence` (collision handling: always spawn), `Initialize` it with the forward GroupSpawner, faction and wave settings, then `Activate`. Verified: after stage 1 the attacker respawned at the forward spawners.

## Objectives verified in game

- **Bar text:** on each stage's `On Started` (server), `TBLGameState.SetObjectiveContextText` with a message per faction and `IsValid` set, as the inspected retail TO map does. Without it the bar had no title.
- **Hold an area:** retail `BaseCaptureArea` spawned at runtime, `ActivateTeamCapturePoint` at stage start, its capture tick bound to `AddScore` (+60 for the attacking team, -30 otherwise, the values retail level scripts use). Fill time is roughly Max Score / 60 ticks.
- **Escort:** your own replicated cart Blueprint that, while its stage runs, counts living attackers and defenders in a box, moves only with at least one attacker and no defender, and adds the stage's Max Score at the end.
- **Damage:** retail `Breakable_Crate` spawned at runtime, its `StatsComponent.OnKilled` bound to `AddScore(1)` with Max Score equal to the crate count. A spawned retail fire pit rack handed out torches that burned the crates.
- **Timeout:** a stage running out goes to its Timeout stage, here the defender victory.

Spawn retail objective Blueprints (capture areas, breakables, weapon racks, siege engines) at runtime from your own actor, for example at placed target points. Placing them in the map cooks the SDK's reconstructed stub components instead of the game's. Some SDK siege stubs log Blueprint errors when loaded, which can fail a cook.

Capture-area tuning lives on its `TeamCapturePointComponent` (`SetTeamCapturePointProperties`: tick interval, capture/neutralize time, players needed, allowed factions, auto-neutralize, immunity times). This is static evidence from SDK declarations and retail data; retail defaults are not the SDK stub values. In a single-player probe the area kept scoring for its owner after everyone left; static evidence suggests auto-neutralize was missing (untested); retail maps inspected enable auto-neutralize with zero immunity. Re-test before relying on any of these settings.

## Authoring limits (UE 4.25 Python)

Editor Python in 4.25 cannot add an interface to a Blueprint or reach the level script Blueprint. Do those steps by hand in the editor or through your own editor C++ module. `GetActorOfClass` is impure in 4.25; unwired exec pins get it pruned. Compile every Blueprint, including the level Blueprint, and re-validate the saved map in a fresh editor process. See [Editor automation](../editor-and-scripting/editor-automation.md).

## Shared objective pieces (proposal)

Retail maps wire each objective by hand in their level script; every objective ultimately calls `AddScore` on a stage. A reusable approach is a small set of placed objective Blueprints sharing one contract: a reference to the stage they score, per-faction bar texts, arming on the stage's start (server only), and replication of what clients see. This design is only partly tested, in single-player probes. In one probe, breaking a breakable actor that had been hit by a thrown weapon crashed the game (one game did not crash; no symbols). Best guess, not confirmed: destroying the actor in the same frame it scored; retail breakables hide and disable collision instead of destroying. That fix is untested.

## Not tested

Several players, bots, dedicated servers (on clients `HasAuthority` can be true for non-replicated placed actors; prefer an explicit server check there), message-table banners, retail battering rams, capture-area settings, crates broken by weapons, and defenders on a touch trigger.
