# Reviewing PAK content and runtime behavior

A PAK is an asset container, not a safety guarantee. Blueprints can invoke native functions already exposed by the shipped game, and packages can attempt to shadow existing assets. Merely placing a native DLL in a PAK does not establish that it will execute. The complete shipped callable surface was not audited by these experiments.

## Review the whole archive

Inventory every entry and mount path, custom dependency, loaded map and potential mod actor. Visible mod-menu entries do not describe everything in the archive. Inspect convention-based actor paths as well as marker metadata; see [first mod](../getting-started/create-first-mod.md).

Separate these findings:

1. A name or capability is present in serialized data.
2. A valid call is connected to executable logic.
3. A loading path can instantiate that logic.
4. Instrumented execution reaches the call in the shipping game.
5. An observable external effect actually occurs.

An asset replacement attempt can fail because of mount precedence, cached assets or incompatible cooked references. A failed local attempt does not prove the archive has no conflicting content. Conversely, a suspicious string alone does not prove execution.

## Bounded shipping observations

`LaunchURL` opened an external browser in a tested shipping build. Such requests can disclose ordinary browser/network metadata to their destination. It is not evidence of arbitrary native execution or silent installation of another PAK.

An instrumented `TBLWebWidget.BrowseToUrl` call reached and returned in a tested shipping build without an observed request. That distinguishes a call with no observed effect from logic that never ran. It does not prove the underlying implementation is universally disabled, nor justify treating it as a confirmed silent network channel.

Stock Blueprint file/network facilities inspected in these experiments did not demonstrate arbitrary PAK self-installation. Game-specific native interfaces were not exhaustively enumerated; do not turn that limited negative result into a general impossibility claim.

## SaveGame files

Any Blueprint mod can write files through the stock SaveGame functions (`SaveGameToSlot`, `SaveDataToSlot`, `DeleteGameInSlot`). This is not specific to structs or any other data type: structs only organize what is saved. In a tested shipping build launched through Unchained, the files went to `%LOCALAPPDATA%\Chivalry 2\Saved_Unchained\SaveGames\<SlotName>.sav`, the same folder Unchained's own mod loader uses for `ModLoader.sav`.

What this allows, from UE 4.25 source inspection (not tested in Chivalry 2):

- The generic save system builds the path as `<Saved>/SaveGames/<SlotName>.sav` without sanitizing the slot name (`Engine/Source/Runtime/Engine/Public/SaveGameSystem.h`, `FGenericSaveGameSystem::GetSaveGamePath`). A slot name containing `..` or path separators could therefore create, overwrite or delete `.sav` files outside the SaveGames folder, wherever the user's account can write. Whether Chivalry 2 replaces this generic implementation was not checked.
- The extension is always `.sav` and the content starts with the engine's save header, so this does not by itself place an executable or a config file. The realistic harms are data ones: overwriting or deleting other mods' saves (including the mod loader's), filling the disk, or leaving stray files.
- Anti-cheat behaviour toward mod-written save files was not documented or tested here. The files are written by the game's own code, as any game save is; treat that as an observation, not as a statement about anti-cheat policy.

When reviewing a PAK, look for SaveGame calls and check their slot names: fixed, simple names are normal; slot names built from strings containing `..`, `/`, `\` or drive letters, or large write loops, deserve scrutiny.

## Safe validation

Use isolated test installations, loopback endpoints and explicit execution checkpoints when investigating behavior. Record game/tool versions and distinguish source inspection from live results. Do not rely on development-only `Print String` for shipping instrumentation.

Static reviews should state capability, reachability and uncertainty separately. No scanner result described here certifies a mod as harmless.
