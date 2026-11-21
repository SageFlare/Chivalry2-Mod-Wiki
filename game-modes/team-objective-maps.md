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

### How retail maps move spawns (static evidence)

Read from one retail Team Objective map's level script and placed actors, plus game-mode defaults; a wider static audit saw the same deactivate, spawn, initialize, activate shape on all retail Team Objective maps. None of this was tested in game. Method: [Reading retail map scripts](../game-research/level-script-research.md).

For one faction, a retail spawn change:

1. Deactivates every `SpawnSequence` of that faction.
2. Spawns, initializes and activates the new sequence. Respawning players use the new spawners at once.
3. Activates the faction's new combat-zone boundary volumes and deactivates the old ones.
4. Waits: 15 s for defenders, 30 s for attackers on the inspected map (20 s for some last-stage defender sets).
5. Deactivates the faction's old out-of-combat (spawn safe) zones and calls `StartForwardSpawning` on a dedicated forward spawner with every character of that faction still out of combat, moving players who stayed in the old spawn room.

The wait is a grace period before that move, not a delay before the new spawns go live. Most stages switch both factions when the stage starts; some switch the attackers 15 or 30 s later. Mid-stage switches come from trigger volumes or objective events.

Observed settings:

- Every sequence: one simultaneous wave, looping, always keep one wave active, fast first spawn, wave timing set to half of the interval (effect is native), no wave parameter overrides. Intervals: attackers 5-10 s, defenders 7-20 s.
- Intervals change at runtime with `SetSecondsBetweenWaves`. On a capture stage: at the start attackers 10 s / defenders 14 s; after the attackers capture, 9 / 11 s; after the defenders retake, 5 / 13 s; with two minutes left the defender interval becomes 20 s.
- `StagedGameMode`: `MinRespawnDelay` 3 s (FFA mode: 5 s). Its seconds-between-waves player-count curve adds 0 s at 0 players, 1.25 s at 32, 2.5 s at 48 and 5 s at 64; whether it applies when a sequence passes no curve was not established.
- Run-in spawners: 3 s prespawn, 3 s autorun sprint, 1.75 s before vulnerable. Retail spawners mostly have 4 x 2 slots.
- Cavalry uses separate spawners, started on a timer and only when horses are enabled for the match.

How interval, half timing, minimum respawn delay and prespawn combine into a player's actual wait is native behaviour; measure it before copying values.

## Objectives verified in game

- **Bar text:** on each stage's `On Started` (server), `TBLGameState.SetObjectiveContextText` with a message per faction and `IsValid` set, as the inspected retail TO map does. Without it the bar had no title.
- **Hold an area:** retail `BaseCaptureArea` spawned at runtime, `ActivateTeamCapturePoint` at stage start, its capture tick bound to `AddScore` (+60 for the attacking team, -30 otherwise, the values retail level scripts use). Fill time is roughly Max Score / 60 ticks.
- **Escort:** your own replicated cart Blueprint that, while its stage runs, counts living attackers and defenders in a box, moves only with at least one attacker and no defender, and adds the stage's Max Score at the end.
- **Damage:** retail `Breakable_Crate` spawned at runtime, its `StatsComponent.OnKilled` bound to `AddScore(1)` with Max Score equal to the crate count. A spawned retail fire pit rack handed out torches that burned the crates.
- **Timeout:** a stage running out goes to its Timeout stage, here the defender victory.

Spawn retail objective Blueprints (capture areas, breakables, weapon racks, siege engines) at runtime from your own actor, for example at placed target points. Placing them in the map cooks the SDK's reconstructed stub components instead of the game's. Some SDK siege stubs log Blueprint errors when loaded, which can fail a cook.

Capture-area tuning lives on its `TeamCapturePointComponent` (`SetTeamCapturePointProperties`: tick interval, capture/neutralize time, players needed, allowed factions, auto-neutralize, immunity times). This is static evidence from SDK declarations and retail data; retail defaults are not the SDK stub values. In a single-player probe the area kept scoring for its owner after everyone left; static evidence suggests auto-neutralize was missing (untested); retail maps inspected enable auto-neutralize with zero immunity. Re-test before relying on any of these settings.

## More objective mechanics (verified in game, 2026-11-12 to 11-15)

One player, local Unchained match, shipping game; retail behaviour read from the game's own Blueprints and data.
- **Bots follow AI tags, not move orders.** A retail `BaseCaptureArea` AI-tags itself for all factions when activated
  (`bAutoAITag`, default true), and the CapturePoint behaviour then pulls every bot, defenders included, into the area. A periodic
  `MoveToLocation` loses to the bot's own decisions (retail level scripts never call it). Bots that must hold a post stayed there
  once the area was tagged for the attacking faction only (`bAutoAITag` false, then `TBLAILibrary.AITagActor(area,
  AI.ObjectiveActor.CapturePoint, attacking faction)`, untagged when its stage ends) and their own base behaviours
  (`B_WalkAroundObjectives`, `B_GoToCombatZoneSafeSpot`, `B_RevivePlayer`) were weighted 0. Never change tag or item override
  behaviours: they are likely one shared instance per faction. Retail steers bots per stage with `DestinationGeneratorVolume`s tagged
  per faction.
- **A capture that is only part of a stage needs its own bottom bar.** Retail links a capture's bar to the stage only when the capture
  is the whole stage. Showing the capture's own contribution stopped its bar from moving with score from the stage's other objectives.
- **Live health for a one-breakable stage = the stage bar.** Retail sets the stage score from the target's health on each hit
  (`trunc((1 - health %) x 1000)`), capped one below the maximum; the kill completes the stage. The bar you see while aiming at a
  breakable may lag until you look away and back, in retail maps as well.
- **Who may take torches and burn:** retail limits both sides with literal factions. The fire pit's weapon rack gets
  `SetUseableByTeam(faction)` and hands out a faction torch (`Carryable_Torch_AgathaOnly` / `_MasonOnly`, which the other faction
  cannot pick up), and the target gets `StatsComponent.SetAllowedDamagingFaction(faction)`. In game: defenders got no torch and could
  not damage tents or the barricade; attackers could. Retail burn markers filter on the exact torch class. Faction getters at
  BeginPlay are untested; pass literal factions as retail maps do.
- **Moving-objective sounds:** post the Play / Stop pair of a looping move sound on a persistent `AkComponent` of the moving actor
  (`AkComponent.PostAkEventByName`; `PostAkEvent` does not compile against the SDK stub unless its callback pins are wired). Posting
  by name on the actor and stopping with `StopActor` played only on the first push.
- **Objective banners without chat:** `BroadcastLocalizedObjectiveMessage` with only `HeaderMessage` filled shows the on-screen banner
  and writes nothing to chat.
- **Moving objectives do not push players by themselves.** UE 4 has no UE3-style mover push: a cart moved without sweep passes
  through a player standing still, and a `PushingComponent` alone does nothing on a plain actor (it needs a driver such as a
  `PushableComponent` or character movement). Retail moving objectives use the native `PushableComponent` on a `PushableSplineActor`
  track.

## Pushables, rams and siege hits

Native pushables, the retail battering ram and siege-engine hits on breakables (all verified in game) are in [Pushables, rams and siege hits](pushables-rams-and-siege.md).

## Player score events (verified in game, 2026-11-16)

Player score (the points pop-ups and scoreboard) is separate from the stage score (`AddScore`). Retail objectives award it from one
data table, `/Game/GameModes/GameScoringInfo/GameScoringDataTable` (not in the SDK, so award by row name). Two retail calls do it:
`TBLSystemLibrary.AwardScoreToPlayers(players, RowName, ...)` pays the row's value, and a `TBLGameScoringComponent.AwardScore`
(override -1 = row value) pays from the component's row. Retail never calls `AwardOverrideScoreToPlayer`.

Retail rows and points in Team Objective (static evidence from the table and retail Blueprints):

| Event | Row (points) | Retail mechanism |
| --- | --- | --- |
| Set fire to a burnable | `Obj_BurningApplied` (100), once | burnable component, first flammable hit, to the torch holder |
| Damage a gate or barricade | `Obj_BreakableDamaged_Percent` (50; `_25`, `_10` variants) | per hit: trunc(points / 100 x damage) to the damage instigator; crates have it off |
| Capture | `Obj_Capture_IntervalBonus` (10) every 2 s, plus a captured bonus | native, when the capture area's `bAwardCaptureInterval`, `CaptureIntervalScoreInterval` and `bAwardScoreWhenCaptured` are set |
| Push a pushable | `Obj_PushableBonus` (10) per interval, `Obj_PushableCompleteBonus` (200) | native pushable |
| Kill a noble / NPC | `Obj_NobleKilled` (100), `Obj_KillNPC` (50) | native |
| Other | `TowerDestroyed`, `WagonDestroyed` (100), `LootSecured` (75), `ConstructionEvent` (250), `RaiseLowerConstruction` (50), `StructureDestroyed` (50) | level scripts or the using player |
| Stage and match | `Obj_StageCompleteBonus` (200), `Match_WinBonus` (750), `Match_LossBonus` (300) | native |

Checked in game with objectives built from your own Blueprints: points for burning tents (torch holder, once), pushing a hand-made
cart (an interval award while it moves, a completion award at the end), damaging a barricade per hit, capturing an area spawned at
runtime with the capture-award flags set before activation, and killing an AI target (one 100-point noble kill, no second native award).
Stage and match bonuses came from the game itself; do not add them again. Award only on the server, only while the objective's stage
runs, and only to players of the intended faction.

## Authoring limits (UE 4.25 Python)

Editor Python in 4.25 cannot add an interface to a Blueprint or reach the level script Blueprint. Do those steps by hand in the editor or through your own editor C++ module. `GetActorOfClass` is impure in 4.25; unwired exec pins get it pruned. Compile every Blueprint, including the level Blueprint, and re-validate the saved map in a fresh editor process. See [Editor automation](../editor-and-scripting/editor-automation.md).

## Shipping shared objective pieces

Never ship the same asset path in two mod PAKs (the last mounted wins, so one map gets the other's version). For a release, copy shared
objective Blueprints under the map's own folder and point the map at the copies (byte copy keeps user-struct member GUIDs; redirect
for one editor session and re-save; save user structs by path, the generic loaded-asset save refuses them), then scan the cooked
files for any reference to the shared folder.

## Shared objective pieces (proposal)

Retail maps wire each objective by hand in their level script; every objective ultimately calls `AddScore` on a stage. A reusable approach is a small set of placed objective Blueprints sharing one contract: a reference to the stage they score, per-faction bar texts, arming on the stage's start (server only), and replication of what clients see. This design is only partly tested, in single-player probes. In one probe, breaking a breakable actor that had been hit by a thrown weapon crashed the game (one game did not crash; no symbols). Best guess, not confirmed: destroying the actor in the same frame it scored; retail breakables hide and disable collision instead of destroying. That fix is untested.

## Not tested

Several players, bots, dedicated servers (on clients `HasAuthority` can be true for non-replicated placed actors; prefer an explicit server check there), message-table banners, retail battering rams with bots or several riders, capture-area settings, crates broken by weapons, and defenders on a touch trigger.
