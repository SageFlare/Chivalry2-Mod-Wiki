# Server hosting and travel

Start with the [Unchained setup guide](https://unchained.wiki/server-hosting/setup-guide/), [configuration reference](https://unchained.wiki/server-hosting/configuration/) and version-matched [release notes](https://github.com/Chiv2-Community/UnchainedLauncher/releases). This is a historical development checklist, not a current hosting-provider recommendation.

## Establish a baseline

1. Select a working game installation and create a small password-protected server on a stock map, initially without mods.
2. Use the selected server profile's INI-folder control where available. Stop the server before manual edits and merge existing sections rather than duplicating them.
3. Match game/ping ports across launcher, host firewall and router/provider firewall. Historical defaults were UDP 7777 and 3075; verify the configured values. Keep administration/internal-query services private.
4. Test same-machine or LAN joining, then have an external client test if internet access is intended. LAN success does not establish public reachability; CGNAT and double NAT can prevent inbound connections.
5. Test spawning, respawning and map changes before adding content. Then enable required per-server mod actors and ensure clients have matching dependencies.
6. Back up configuration and version records; verify restart behavior and remote-session lifetime before unattended hosting.

An arbitrary hosted Windows machine is not automatically compatible. Verify networking and launcher requirements before committing to a provider. Regular managed game servers and Unchained hosting are different services; do not assume custom PAK or launcher access.

## Version-specific observations

The following were recorded with launcher **1.0 RC4** in October 2026 and may change:

| Observation | Practical consequence |
| --- | --- |
| RCON `servertravel <Map>?Key=Value` carried map options successfully | Test the final travel address rather than assuming process startup options survive |
| Startup moved from frontend to a fresh map address | Additional process arguments did not necessarily reach the selected map |
| An `=` in the selected start-map value failed argument parsing | Inspect startup logs and version-specific parser behavior |
| Rotation entries containing valued options stalled a tested transition | Validate rotation transitions separately from direct travel |
| Bare options persisted into a later plain map entry | Explicitly test reset/default semantics between maps |

These observations do not prescribe any custom rotation helper. Use supported launcher/RCON facilities and verify each route in the chosen version.

A headless host's own player body was observed at match start on a stock FFA map. `IdleKickTimerSpectate` subsequently moved it to spectator; lowering that timer affects every idle player. RCON `SwitchToSpectator` moved the host player in the tested setup. Confirm behavior before automating it.

Older guidance about matching host/client FPS did not describe the tested patched setup. Consult the selected desynchronization patch's release notes and mode limitations rather than copying historical FPS advice.
