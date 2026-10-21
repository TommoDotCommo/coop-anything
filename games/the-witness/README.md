# The Witness (2016)

| | |
|---|---|
| Engine | In-house engine (Thekla). Family not confirmed: `detect_engine` matched no signature |
| Versions tested | Steam build `11370412` (app `210970`), `witness64_d3d11.exe` SHA-256 `8d672d444df6a6df7130f25a517bdab1af92731fe2138dc5b324d519561d55a5`, Windows |
| Projects | none found. Related tooling: [witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) (MIT), [witness-trainer](https://github.com/jbzdarkid/witness-trainer) (MIT) |
| QA level | none |
| Last updated | 2026-10-22 |

## Where things stand

There is no published co-op for The Witness, and no mod was found. A prototype now exists on
one PC: two copies of the game exchange camera pose over local UDP, and each draws the other
as a puppet through a Direct3D 11 overlay. Shared puzzle state is not done: the solved flag
can be read, but there is no safe write path yet. Tested on one machine, localhost only, by
hand plus a scripted walk test (see QA).

## Existing work

No co-op or multiplayer project. Searched and not found: this repo, GitHub issues, PRs
and repository search, web search. Not checked: Nexus, ModDB, Reddit, archive.org, game
and speedrun Discords, non-English communities (see the directory entry's `searched:` list).

Tooling that reaches into the game:

| Project | License | Last push | What it does |
|---|---|---|---|
| [witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) | MIT | 2026-01 | Reads and writes entity fields (panels, doors) from outside the process |
| [witness-trainer](https://github.com/jbzdarkid/witness-trainer) | MIT | 2026-09 | Camera position and angle, noclip, save control, panel state, found by signature scan |
| Archipelago `worlds/witness` | see repo | n/a | Randomizer client runs beside the game; how it hooks the game was not read |
| [TheWitnessExplorer](https://github.com/ClementSparrow/TheWitnessExplorer) | none stated | 2021 | Python asset exploration |

## How the game works (for co-op)

Observed on build 11370412 (details and sources in [NOTES.md](NOTES.md)):

- Single-player, no netcode found. Assets are in `data-pc.zip`.
- An external process can read the game: a global entity table (`GLOBALS` is `0x62D0A0`
  from the module base) holds a pointer array indexed by entity id; fields sit at fixed
  offsets. Addresses are found by signature scan, as the trainer does.
- Camera position (3 floats) and angles (2 floats) are at static addresses and followed
  the player in a live run.
- Panel solved state: a float at entity `+0x294` (`1.0` when solved) and an int at `+0x298`
  ("ever solved"). Both flipped when one panel was solved; checked once, on one panel.

- A proxy `AnselSDK64.dll` (the exe loads it from its own folder) loads our code into the game.
  It hooks `IDXGISwapChain::Present` by patching the shared swap-chain vtable and draws screen
  quads at world positions. Camera model: z up, `forward = (cos(phi)cos(theta), cos(phi)sin(theta), sin(phi))`,
  vertical field of view about 60 degrees; checked against screenshots at two headings and while walking.
- Moving a panel entity's position field does not move it on screen, so a puppet cannot be an existing panel.
- Launch flags `-skip_config_dialog -windowed -new_game` start the world with no input.

Not known: how to safely write a solved panel and open its door or cable, how saves are
written, and whether other store builds differ (the randomizer already knows two builds
with different offsets).

## What we changed

Nothing in the game files; the real install was never modified. In a disposable copy we added a
proxy DLL (pose exchange over UDP, puppet overlay). Its source is not published yet and will
live in its own repo.

## QA

No automated suite yet, so QA level stays `none`. Checked on one machine (Windows, Steam build
11370412): read-only memory probes before and after solving one panel; a scripted walk (a keypress
sent to one copy) after which the other copy's log showed the first copy's position
(-199.82, -119.82, 3.73) against its real camera (-199.819, -119.816, 3.727), 31 ms old, plus
screenshots of the puppet. Not tested: a real network, packet loss, more than two players,
fullscreen, non-16:9 windows. The probe and test scripts are not published yet.

## How to play

Not playable together yet (prototype only, two copies on one PC).

## Known issues

Test hazard for the next agent: the game keeps saves in `%APPDATA%\The Witness` whatever
folder the exe runs from. Starting a New Game from a copied install rewrote the owner's
latest existing save and settings file there. Back up and hash that folder first.

## Next steps

1. Work out a safe write path for solved state (and sample the state and traced-edge fields, `+0x280` and `+0x228`) so one player's solve opens the other's door.
2. Real networking (Steam relay or direct IP) with sequence numbers, then loss and lag runs.
3. Occlusion and a real puppet model instead of a flat rectangle.
4. A bot that drives a panel solve with a state diff, then raise the QA level honestly.
5. Publish the mod in its own repo; read the Archipelago client for another view of the same hooks; finish the unchecked searches.
