# Mod audio and the volume sliders

How a mod's sounds can follow Chivalry 2's volume sliders. This applies to any mod that plays sound. Evidence levels are given per claim: static inspection of the game's data and SDK declarations, and shipping-game tests (one player, 2026-11-08, UE 4.25.4 ArgonSDK).

## Why ordinary UE4 sounds ignore the sliders

Chivalry 2's own audio uses Wwise 2019.2.6 (SoundBank version 135). Static inspection: the slider values (Master, Game and so on, 0-100) live in the cloud user settings reached from `TBLGameUserSettings`. Applying them sets global Wwise RTPCs (`MASTER`, `GAME`, `AMBIANCE` and others), which drive bus volumes defined in the game's Init bank. The apply step broadcasts no delegate a mod can bind for audio.

A mod's SoundCues and SoundWaves play through UE4's legacy audio path, not Wwise, so the sliders do not affect them. Verified in game: with Master at 0, a mod's ambient wind was still audible.

## Recommended: ship a Wwise bank

Verified in game: a mod-supplied bank loaded and played, and its looping tone on an ambient bus followed the Master and Game sliders live, including while the options menu was open, like the game's own sounds.

- Build one bank in the v135 format. Route sounds to busses named as the game's: a Wwise bus ID is the 32-bit FNV-1 hash of the lowercase bus name, so identically named busses resolve to the game's busses (checked statically for several, including `MASTER`, `GAME`, `AMBIANCE`). The test bank used the same project ID as the game's banks.
- Prefix every event, sound and bank name with something unique to your mod so its IDs cannot collide with the game's or another mod's.
- The game loads banks by name, not through event-based packaging. Pack the `.bnk` as a loose file in the PAK at `TBL/Content/WwiseAudio/Windows/<Bank>.bnk`, alongside your cooked content. Add it to your PAK response file explicitly; a normal cook does not produce it. See [Cook and package](../packaging-and-hosting/cook-and-package.md).
- In Blueprint: `AkGameplayStatics.LoadBankByName`, then `PostEventByName`, `PostEventAtLocationByName` or an `AkComponent`'s `PostAkEventByName`. Unloading the bank on EndPlay is a sensible cleanup; its effect was not tested separately.

Not verified: whether routing directly to the master bus behaves differently from an ambient bus (the menu has no ambient slider to tell them apart), and attenuation sharesets, random/sequence containers and 3D positioning in hand-built banks. Test each before relying on it.

### Authoring and legal caveat

There are two ways to produce a compatible bank:

- **Official tool:** Audiokinetic's Wwise authoring tool, version 2019.2.6 to match the game runtime. Its free licensing is limited to non-commercial use and has asset limits and registration terms; read the current licence yourself.
- **Writing the format directly** from public, community-written bank parsers, without the Audiokinetic tools. This is how the tested bank was produced. The format knowledge comes from community reverse engineering, which is a legal grey area; Wwise's licence forbids reverse engineering for its licensees.

This is not legal advice. Confirm current community and launcher policy on shipping custom banks before publishing.

## Fallbacks for UE4 audio

If a sound must stay on UE4 audio, scale it yourself with a SoundMix class override driven by the slider values:

- **Polled** (re-read the volume settings every 0.5 s and push the override): works in game. The cost is negligible because UE 4.25 ignores a set to an unchanged value.
- **Event-driven** (bind the options properties' value-changed event and the user-settings UI update event): partial in game. UE4 audio is paused or ducked while the menu is open, and the new volume applied only after the menu closed.
- **Not possible from Blueprint:** routing UE4 audio into Wwise. `AkAudioInputComponent` is abstract and needs a native sample-buffer implementation.

Related: a sound placed on a level-sequence audio track (for example a prologue voice line) is UE4 audio and is very likely affected the same way (inferred, not tested); see [FFA map setup](../game-modes/ffa-map-setup.md).
