# How bots choose objectives

[Game modes](README.md) · [Agent index](../AGENT-INDEX.md)

How Chivalry 2's Utility AI bots decide what to do, and how the official Team Objective maps steer them. Evidence: the retail AI assets (`/Game/AI/Utility/...`: behaviour sets, behaviours and the `OverrideBehaviorSets` table, read as JSON dumps), the level scripts of 13 official TO maps (bytecode), and the SDK headers. This is static evidence; it has not been checked in game unless a line says so.

## Scoring

Each bot runs two decision makers: a general one (base sets `BS_MeleeFighter` and `BS_General`) and a combat one (attacks, blocks, feints, ripostes). A behaviour has an integer weight, a list of considerations (an input normalised between a minimum and a maximum, then a curve), one target generator and one action. The data fits the usual utility-AI shape: score = weight × product of the consideration curves, highest score wins, and the running behaviour gets about ×1.25 so bots do not flip-flop.

Approximate weights, highest first: forced location 10000; siege engine actions and dropping items 100-150; leaving an out-of-bounds combat zone 25; mounting a ram, siege engine or picking up a petard 20; a bomb 17; **engaging an enemy 15**; attacking a breakable 10; walking to a breakable, using a ramp, throwing a torch 8; picking up a carryable or loot 7; **capture point 6**; **picking up a torch, walking to a burnable, escorting 5**; reviving 4; **walking around objectives 2**.

So a perceived enemy almost always wins. Objectives win when no enemy is near, and roaming is the floor.

## AI tags switch behaviours on

The `OverrideBehaviorSets` data table maps a trigger to a behaviour set. Two kinds of rows:

- **Tag rows add a set.** An actor tagged `AI.ObjectiveActor.<Kind>` for a faction gives that faction's bots the matching behaviours. Tags are set with `TBLAILibrary.AITagActor(Actor, Tags, Faction, bUntagWhenActorDies)` and removed with `AIUntagActor`, `AIUntagAllActorsForFaction` or `ClearAllAITagsForFaction`. The faction argument is the faction whose bots react.
- **Held item and mount rows replace the general sets** while the bot holds the item or rides the engine (torch, carryables, ram, siege engines, petards).

`EFaction` values: Agatha 0, Mason 1, FFA 2, None 3, All 4, Tenosia 5.

| Tag | What bots do | Needs an interaction |
| --- | --- | --- |
| `Breakable` | walk to it and melee it (vertical reach about 2 m) | no |
| `Escort` | stand at six points beside it | no |
| `CapturePoint` | walk to grid points inside its bounds | no |
| `Burnable` with `TorchPit` | fetch a torch from a tagged pit, walk to a throw point, throw from about 4 m | yes (the pit) |
| `Bomb` | hold-use it and patrol around it (the only patrol-style behaviour) | yes |
| `BatteringRam`, `LoadableSiegeEngine`, `InteractableRamp`, `RampPiece` / `ConstructableRamp`, `Carryable` / `Dropzone`, `Petard` / `PetardDropzone`, `Lootable`, `TopplingStone` | mount, load, build, carry or use the object | yes |
| `AI.RallyPoint` | go to forward rally points within about 60 m | no |
| `AI.ForcedLocation` | go there; overrides the faction's whole behaviour set; no official TO map uses it | no |
| `AI.DestinationVolume` on a `DestinationGeneratorVolume` | roam the volume's saved points | no |

There is no lever behaviour and no "defend" behaviour.

## Why bots rarely burn objectives

Picking up a torch is gated by the time left in the objective: roughly ×0.2 with four minutes or more left and ×1 only with two minutes or less. Early in a stage its best score is about 1, below roaming (2) and far below fighting (15), and it drops further when other bots already fetch torches. Expect bot torch runs only near the end of a stage and only when no enemy is near. A bot holding a torch loses its general behaviours and only walks to a throw point.

## How official TO maps steer bots

1. **Every stage has a roaming volume.** One `DestinationGeneratorVolume` per stage (two for attacker and defender variants or for lanes), usually tagged `AI.DestinationVolume` for All at stage start, or switched with `Activate()` / `Deactivate()`, and cleared at stage end. Volumes sit on the objective, are typically 40-130 m wide with a destination density of 600-1400 uu and 30-100 saved points. A moving objective gets a chain of volumes swapped as it advances.
2. **Objective tags by kind.** Objectives the attackers act on are tagged for the attacking faction only: breakables, burnables and their torch pits, petards, carryables, ramp pieces, toppling stones, siege engines. Contested objectives are tagged for All: capture points (a capture area tags itself when `bAutoAITag` is on, the default), escort on pushables and on VIPs to kill, bombs and cranks.
3. **Burn stages always use the same recipe:** a roaming volume for All, `Burnable` on the targets and `TorchPit` on the fire pits, both for the attackers.
4. **Defender-specific steering is rare** and done with volumes: a separate defender volume placed nearer the objective next to the attackers' volume, or a rally point on an escorted VIP.
5. **Cleanup:** targets are untagged as they are destroyed (`bUntagWhenActorDies`), the stage volume is untagged at the next stage, or everything is cleared with `ClearAllAITagsForFaction(All)`.
6. **No move orders.** The official TO level scripts never order bots to move. A move order does not hold anyway: the utility AI picks a new behaviour right after it (seen in game).

## Open questions

Whether `Activate()` on a volume tags it for every faction; the official default range of the distance input used by roaming (whether bots prefer near or far points); whether bots added with the server's add-bots command react to the same tags as map-spawned bots.
