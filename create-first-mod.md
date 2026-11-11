# Create a first Blueprint mod

This is an introductory recipe distilled from SDK documentation and implementation review. The complete tutorial was not independently run end to end. Use the [Field Guide](https://knutschbert.github.io/ArgonSDK-FieldGuide/) alongside it and validate in your own game version.

## Setup

Open a working ArgonSDK `TBL.uproject` in UE 4.25.4. Install its required dependencies and compatible compiler if project modules need building. Basic Blueprint content does not require extracting the entire game art library. Custom skeletal content may require a different engine workflow; check the SDK instructions.

Create assets directly beneath `<ProjectRoot>/Content/Mods/AgMods/ExampleMod`. Their package root is `/Game/Mods/AgMods/ExampleMod`. An external source folder mapped by a directory junction is optional; keep one authoritative editable copy and verify the junction target before using it.

## Actor and marker

1. Create a Blueprint subclass of `ArgonSDKModBase` named `ExampleMod` in that folder.
2. Create a `DA_ModMarker` data asset and add the actor class to its Mod Actors collection. Save both assets.
3. For a local display experiment, allow offline loading and clientside execution; do not enable frontend-only or host-only behavior unintentionally. Review the loader flags in your SDK version.
4. In BeginPlay, preserve the parent call and invoke `Show Local Chat` with a distinctive message.
5. Compile and save. Reopen in a fresh editor process before packaging.

The inspected loader resolves an enabled mod actor by the convention `/Game/Mods/AgMods/<Name>/<Name>`. Marker metadata supports discovery and UI; **marker presence alone is not the runtime spawn trigger**. The actor still needs to be selected or requested by the applicable loading path. A loose Blueprint in a PAK does not automatically execute.

## Iterate

Follow [cook and package](cook-and-package.md), ensuring the actor, marker and required companions are included. Install into the game installation used by Unchained, enable the mod through the version-appropriate loading controls, and enter a match. Check the message, change it, rebuild and verify that the new message appears.

`Print String` is unsuitable as the sole shipping-game checkpoint. A successful editor preview does not establish discovery, packaging or runtime execution.

If nothing appears, check the actor path, marker reference, loading selection, BeginPlay chain, loader flags, archive contents and duplicate old PAKs. Keep client-only display tests separate from authoritative gameplay and replication tests.
