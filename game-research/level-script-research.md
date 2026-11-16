# Reading retail map scripts and placed actors

How to read what a shipped Chivalry 2 map does (stage logic, spawn changes, objective wiring) without running the game. This is static evidence: it shows what the level script calls and how placed actors are configured, not how native C++ code then behaves. Measure timing-sensitive behaviour in game before relying on it. Worked result: [how retail maps move spawns](../game-modes/team-objective-maps.md#how-retail-maps-move-spawns-static-evidence).

## Why not the editor

Retail maps ship as cooked packages, and the UE 4.25 editor used by the ArgonSDK does not open cooked packages. Read them with a package parser instead. The retail data is also the version that actually runs.

## Export

- Use [CUE4Parse](https://github.com/FabianFG/CUE4Parse) (library) or a tool built on it such as [FModel](https://github.com/4sval/FModel). Game version UE 4.25; the Chivalry 2 PAKs were read with an all-zero AES key.
- Blueprint bytecode is only exported when script reading is enabled. In CUE4Parse, set `ReadScriptData = true` on the file provider before loading. Without it the level script's functions have no `ScriptBytecode`. Whether FModel's JSON export includes it was not tested.
- Open the installed `TBL/Content/Paks` folder read-only and write exports elsewhere. Mod PAKs in that folder are mounted too and can override retail files: check which PAK serves each file you dump (the retail Team Objective maps resolve to `pakchunk0`).
- A map's gameplay actors may sit in sublevel packages rather than in the persistent map.

## Reading the level script

The level Blueprint compiles into one large function, `ExecuteUbergraph_<Map>`. Each event is a small stub that jumps into it at an offset.

- Offset order is not execution order. Follow `goto` jumps from the event's entry offset.
- Sequence nodes compile to a flow stack: `push N` queues an output, `pop` continues with the latest queued output. A conditional pop ends the current branch when its condition is false.
- `Delay(t, LatentActionInfo{R, ...})` lets the current flow carry on immediately. After `t` seconds execution resumes at offset `R` with an empty flow stack.
- Walking every reachable statement from an event shows what the event can touch, but merges branches and loses order. For order and timing, follow jumps, the flow stack and delay resume offsets. A reliable shortcut is to list every `Delay` with the statements before it and the first statements its resume offset reaches, then read the raw listing of one block to confirm the pattern.
- Looping timers (a `Delay` whose resume path reaches the same `Delay`) make a naive recursive trace never end.
- Gate, DoOnce and ForEach macros expand into many temporary-variable statements around the real calls.
- `ActivateDeactivate(byte)`: 0 activates, 1 deactivates. Faction literals follow `EFaction` (0 Agatha, 1 Mason, 2 Tenosia); check them against the map's attacking and defending faction.

## Placed actors and positions

- An actor's location is on its root component: follow `RootComponent` to the component export (its `Outer` names the actor) and read `RelativeLocation`.
- Spawners also store `SpawnCenter` and one `SpawnComponentData` entry per slot. A `GroupSpawner` lists member spawners; expand it to count slots.
- Cooked class default objects store only overrides. Follow the parent class chain ([inheritance](game-data-research.md#resolve-inheritance)); a value absent everywhere has its native default, which cannot be read statically.
