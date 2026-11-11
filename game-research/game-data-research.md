# Read-only game-data research

These findings concern the inspected Chivalry 2 data and exporter versions. They do not establish current exporter support for every asset.

Use a public parser such as CUE4Parse or an asset viewer such as UModel with version settings appropriate to the game. Keep installed packages read-only and write exports elsewhere. Record package/tool hashes and effective mount order: extra mod PAKs can override the data you think you are examining.

## Resolve inheritance

Cooked class default objects often store only overrides. Follow `ParentClass` chains before interpreting an absent value. Reconstructed SDK native constructor values are useful hypotheses but not authoritative proof of shipping defaults.

In the inspected data:

- Weapon `Abilities` maps extended the global `Gameplay/Abilities` table rather than replacing it wholesale.
- Per-weapon timings appeared in `Gameplay/AbilitiesOverride`. A value of -1 indicated inheritance through the weapon chain and then a default row.
- Row keys could use an ability-row name rather than the attack slot name.
- Per-weapon movement sequences appeared in `Gameplay/DataTables/AbilitiesOverrideAsset`.

Record the complete resolution chain so another researcher can distinguish serialized data from inferred defaults.

## Animation timing

Montage section/notify times depend on `LinkMethod`:

| Method | Interpretation |
| --- | --- |
| Absolute | `LinkValue` |
| Relative | Segment start plus value |
| Proportional | Segment start plus value times segment length |

The tested exporters did not decode the required animation tracks. Montage root-motion windows therefore supported timing research, not exact movement distance. Use shipping-game measurements for distance unless a verified decoder supplies the actual transforms.

When invoking UModel from Git Bash, MSYS path translation can corrupt Unreal-style paths; `MSYS_NO_PATHCONV=1` avoided that issue in the tested environment. It is not needed merely because the tool runs on Windows.
