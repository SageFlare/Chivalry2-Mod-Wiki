# Pushables, rams and siege hits

Moving objectives, the retail battering ram and siege-engine hits for Team Objective maps made as mods. The stage frame (stages, objectives, spawns, winner) is in [Team Objective maps](team-objective-maps.md).

## Native pushables (verified in game, 2026-11-16)

One player, local Unchained match, UnchainedPlugin 1.0.0. Setup: the native `PushableComponent` on your own Blueprint, with a track
that is your own Blueprint child of the native `PushableSplineActor`.

- **Configure it at stage start** with a full `ReConfigurePushable` (properties struct, spline, end points, time), as retail Bulwark
  does, then `ResetProgress(0)`, `ActivatePushable(True)` and `ProceedToNextEndPoint`. A configuration saved only on the placed or
  template component never moved, even with `ResetProgress(0)` first (values equal to the SDK stub defaults are not saved at all).
- Configured that way it moved within about a second when an attacker entered its zone, slowed and stopped when they left, halted
  and resumed at end points, and finished.
- **It pushes a player standing still in its path out of the way**, which a hand-moved cart does not do.
- After `ReversePushableByDistance` it rolled back; once an attacker left the zone and came back, it moved forward again.
- A defender NPC (a spawned AI kill target) in the zone did not contest it. Bot and player defenders were not tested.
- Push score pop-ups appeared.
- **A long converted route works too (2026-11-17):** a track of about 9,200 units (92 m) built from an old map's path (a spline point at least every
  50 units, the path lifted to the terrain, `TimeToComplete` = path length / original speed) moved a caravan with attached wagon,
  crates and spinning wheels over slopes to the end and completed the stage. It floated slightly in places (a per-map touch-up).

## Retail battering ram (verified in game, 2026-11-20 and 2026-11-21)

One player, local Unchained match, UnchainedPlugin 1.0.0-rc4, Unchained-Mods 0.2.2 and 0.2.3. The ram is the retail
`Test_BatteringRam_pushable_Citadel` or `Test_BatteringRam_pushable_LD` (`/Game/Interactables/SiegeEngines/BatteringRam/`), pushed
along your own `PushableSplineActor` child as above.

- **Spawn it at runtime from a string**, never from a class reference: the SDK only has stubs for the ram, and referencing them
  logs Blueprint errors at cook. Load the class default object with `LoadAsset_Blocking` on
  `<package>.Default__<Name>_C` (for example
  `/Game/Interactables/SiegeEngines/BatteringRam/Test_BatteringRam_pushable_Citadel.Default__Test_BatteringRam_pushable_Citadel_C`),
  take `GetObjectClass`, cast it to an actor class and spawn it. Linking the loaded object straight into a class pin compiles (the
  editor inserts a `GetObjectClass` itself), but on a path that names the class it would cast the Blueprint class object's own
  class and fail (engine source reading, not tested). Both variants loaded, followed the track and behaved the same.
- **Recreate the ram Blueprint's own start and end logic.** Its `activate_pushable` event cannot be called without a class
  reference, so you drive its `PushableComponent` yourself, and the Blueprint's other calls are then yours to make. Retail order:
  `ActivatePushable(True)`, HUD marker `SetVisible(True)`, `ProceedToNextEndPoint`, then `SetEnabled(False, False)` on every
  `InteractableComponent` of the ram. On `OnPushableCompleted` it calls `SetEnabled(True, False)` on each again. Without the disable,
  players could mount the ram mid-track; with it they could mount only after it completed. Call `SetUseableByTeam(<attacking
  faction>)` on the same components.
- **The HUD marker stays on after completion** in the retail Blueprint (source inspection): its location function puts it above the ram
  while the pushable is active and on a mount point once it is not.
- **Swinging:** one rider could mount and swing. Each landed swing did 2500 damage to a `StatsComponent` target.
- **Gate health:** a retail gate spawned at runtime from its class has the class default health (`BP_BaseBreakableGate`: 200, one
  swing). Retail maps set health per placed gate (TO_Citadel's main gate: 20000). After spawning, call
  `SetBase(MaxHealth, <hp>)` and `SetBase(Health, <hp>)` on the gate's `StatsComponent`; at 20000 the gate broke after 8 swings. We
  also called `SetAllowedDamagingFaction(<attacking faction>)` on it (the retail gate allows no faction by default; not tested without).
- **Keep the ram's end point short of the gate.** `BP_BaseBreakableGate.OnPushableBeginOverlap` applies the gate's MaxHealth as
  `StandardDamage` when a pushable overlaps it, an instant kill (source inspection; not provoked in game). A gate 495 units past the
  track end, as on retail maps, was hit only by swings.
- **Open points:** spawning the ram with its capsule centre 136 units above the track (the retail Citadel offset) left it in the air
  until it dropped before the stage started; measure the offset for your track instead of copying it. Beside our own target blocks
  the beam rose by itself once at completion and settled before mounting worked; beside the retail gate it did not. Cause unknown.
  Bots, several riders, dismounting and dedicated servers were not tested.

## Siege hits on breakables (verified in game, 2026-11-16)

Targets with a `StatsComponent`, hit by the retail catapult and ballista spawned at runtime:

- Every catapult rock and ballista bolt reported the damage source `RangedDamage`, not `SiegeDamage`. Damage causer = the engine
  (`Catapult_C`, `Ballista_C`), instigator = the player pawn, projectile `Carryable_CatapultRock_C` / `Ammo_BallistaBolt_C`, ability
  row `ShootCatapult` / `ShootBallista`. A rock did 3000 per hit, a bolt 100. To react to one engine only, filter on the damage causer
  class, projectile or ability row, not on the damage source.
- `bDisableProjectileDamage` blocked rocks, bolts and thrown player weapons. Adding `bEnableSiegeDamage` let the siege-engine
  projectiles through again while thrown weapons stayed blocked; melee still hit. `bEnableSiegeDamage` alone changed nothing.
- `AllowedDamageSource` = `[SiegeDamage]` blocked every hit tested (melee, throws, rocks, bolts). Do not use it for catapult targets;
  which weapons do deal `SiegeDamage` (rams?) is unknown.
- A `HitWorld` tag on the target made no difference.
- The retail catapult and ballista classes face their local +X axis.
