# Spawns, respawns and navigation

See [FFA lifecycle](ffa-lifecycle.md) for object roles, request/possession/control boundaries, function signatures and unresolved native behavior.

The following FFA pattern was exercised in custom maps. It requires implementing a startup actor in your own project; no private startup asset is supplied or required.

## Spawn structure

1. Use one-slot `BaseSpawner` actors (`rows=1`, `columns=1`) with wave parameters `override_faction=True`, `faction=FFA`.
2. Reference them through a `GroupSpawner`, with child priority 0, max -1 and `spawner_selection_mode=UNIFORM_DISTRIBUTE`. Assign the group as World Settings `attacker_initial_spawn`.
3. Configure a `SpawnSequence` with those base spawners, faction FFA, and explicit wave timing. Tested values were three simultaneous waves and one second between waves, with the half-wave timing override disabled.
4. When reducing startup delay, set both each timing value and its override flag. Tested values for `seconds_before_spawning_starts`, `minimum_wait_time_seconds` and `prespawn_time` were zero; the joinable window remained five seconds.
5. Implement authority-side startup logic that waits for the match to be in progress before activating the sequence. Blind BeginPlay activation caused pre-match pawn/spectating problems. Inspect the SDK's match-state interfaces and verify the actual shipping sequence.

An initial spawner alone does not establish repeated respawns. Game-mode settings, server configuration and travel options can also control warmup; changing one layer may leave a countdown from another.

This is a limit of the initial-only configuration, not a one-use limit on the spawner class. Match start waits, warmup, preparation, wave cadence, join windows and player delays are separate. A five-second joinable window is not a mandatory player countdown. Use `IsMatchInProgress` for active-play gating; `HasMatchStarted` can remain true after the match ends. A start-event subscription must also handle a match already in progress.

## Placement and bots

Place spawners in the full editor; commandlet placement caused crashes. After rotating a spawner, inspect its cached `SpawnComponentData` transforms as well as the actor rotation. Verify after reopening.

Bot permission may inherit the game default when `override_allow_bots=False`. Confirm the effective setting and provide enough spawn capacity.

Build navigation, wait for it to finish and validate routes using the intended agent dimensions. Check representative connected routes, stairs, narrow gaps, spawn exits and deliberate barriers. A navmesh volume or green coverage alone does not prove path connectivity. Players do not require a navmesh merely to spawn.

Do not require every surface to be walkable. Record intended access and exclusions. Use collision queries and player-sized sweeps for clearance, then test actual bots and players.

## Runtime acceptance

Check initial spawn, death-to-respawn, spectating, match transitions and bot movement separately. Saved spawner/sequence properties establish configuration, not runtime success. Multiplayer adds authority and replication checks beyond a local match.
