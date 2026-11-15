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
  is the whole stage. Showing the capture's own contribution kept the tower bar from moving with unrelated kills.
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
  track. Partial probe: configuring it at stage start with `ReConfigurePushable` (full properties struct, as retail Bulwark does)
  made it move and award push score; values saved only on a placed instance did not (values equal to the SDK stub defaults are not
  saved); after `ReversePushableByDistance` it could not be pushed again without more calls (not solved yet).

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

Several players, bots, dedicated servers (on clients `HasAuthority` can be true for non-replicated placed actors; prefer an explicit server check there), message-table banners, retail battering rams, capture-area settings, crates broken by weapons, and defenders on a touch trigger.
