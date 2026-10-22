# The Witness (2016)

| | |
|---|---|
| Engine | In-house engine (Thekla). Family not confirmed: `detect_engine` matched no signature |
| Versions tested | Steam build `11370412` (app `210970`), `witness64_d3d11.exe` SHA-256 `8d672d444df6a6df7130f25a517bdab1af92731fe2138dc5b324d519561d55a5`, Windows |
| Projects | [witness-coop](https://github.com/TommoDotCommo/witness-coop) (MIT, dev preview). Related tooling: [witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) (MIT), [witness-trainer](https://github.com/jbzdarkid/witness-trainer) (MIT) |
| QA level | automated (narrow: one puzzle, localhost, see QA) |
| Last updated | 2026-10-22 |

## Where things stand

No other co-op for The Witness was found. [witness-coop](https://github.com/TommoDotCommo/witness-coop) is a dev-preview prototype that runs on one PC with two copies of the game:
each player sees the other as a puppet drawn through a Direct3D 11 overlay, and when one player solves a panel the other
game marks it solved and the door it gates opens, started by the game's own door function. Panels and doors are synced
for all 594 panels and 327 doors, but only the first puzzle (the door at the end of the start tunnel) has been played
through and tested. Cables, panels powered by other panels, lasers, plates, bridges and the rest are not synced.

## Existing work

No co-op or multiplayer project. Searched and not found: this repo, GitHub issues, PRs
and repository search, web search. Not checked: Nexus, ModDB, Reddit, archive.org, game
and speedrun Discords, non-English communities (see the directory entry's `searched:` list).

Tooling that reaches into the game:

| Project | License | Last push | What it does |
|---|---|---|---|
| [witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) | MIT | 2026-01 | Reads and writes entity fields (panels, doors) from outside the process |
| [witness-trainer](https://github.com/jbzdarkid/witness-trainer) | MIT | 2026-09 | Camera position and angle, noclip, save control, panel state, a door-open function call, found by signature scan |
| Archipelago `worlds/witness` | see repo | n/a | Randomizer client runs beside the game; how it hooks the game was not read |
| [TheWitnessExplorer](https://github.com/ClementSparrow/TheWitnessExplorer) | none stated | 2021 | Python asset exploration |

## How the game works (for co-op)

Observed on build 11370412 (details and sources in [NOTES.md](NOTES.md)):

- Single-player, no netcode found. Assets are in `data-pc.zip`.
- A global entity table (`GLOBALS` is `0x62D0A0` from the module base) holds a pointer array indexed by entity id.
  Entity types have names reachable from each entity (`entity+0x08` to a descriptor, name at `+0x08`): 48 types,
  among them `Machine_Panel` (594), `Door` (327), `Power_Cable`, `Pressure_Plate`, `Multipanel`, `Human` (the player, id `0x1E465`).
- Camera position (3 floats) and angles (2 floats) are at static addresses and follow the player.
- Panel solved state: a float at entity `+0x294` (`1.0` when solved), an int at `+0x298` ("ever solved"), a state int at `+0x280`.
  A door is open when its power word at `+0x1D0` is `1.0`.
- A solve changes few things in memory (15 entities for the first panel): the panel's own fields, power words on the
  target door and its group node, and the transforms of the door parts. Writing those fields into another game did **not**
  open its door; the opening starts from a function call (`base + 0x8E4B0`, the one the trainer calls).
- Launching: the exe shows a graphics options dialog first. `-skip_config_dialog -windowed -new_game` start the world with no input.
- A proxy `AnselSDK64.dll` (the exe loads it from its own folder) loads our code into the game. It hooks `IDXGISwapChain::Present`
  and draws screen quads at world positions (camera model: z up, vertical field of view about 60 degrees, checked against
  screenshots at two headings and while walking). Moving a panel entity's position field does not move it on screen.

Not known: how to sync cables and powered panels, other entity effects, saves and new games, and whether other store builds
differ (the randomizer already knows two builds with different offsets).

## What we changed

Nothing in the game files; the real install was never modified. In disposable copies we added a proxy DLL with: pose exchange
and puppet overlay, a detector for solved panels and opened doors, a reliable event layer (numbered, acknowledged, resent,
applied once and in order, with epochs and a full resend when a peer restarts), and an applier on the render thread with an
echo guard. The source is in [witness-coop](https://github.com/TommoDotCommo/witness-coop).

## QA

A scripted suite (`tools/qa.sh` in witness-coop) restarts both games, has a bot play player A with real keyboard and mouse input
(walk 46 m, aim from the camera angles, enter the first panel, trace the line), and compares the solved panels and open doors
read from both games' memory. Last run, one machine, localhost, build 11370412:

```
baseline                                 PASS   identical (1 panel, 10 doors); B sent nothing back; no resend needed
lost first send of every event           PASS   identical; A resent all 11 events
MUTANT: no resend + lost first send      PASS   games diverged (panel 0x64 and door 0x34eda only on A), as required
MUTANT: echo guard off                   PASS   B echoed 2 events back, as required
ALL OK
```

Earlier checks: read-only memory probes before and after a solve; a scripted walk after which the other game's log showed the
first game's position (-199.82, -119.82, 3.73) against its real camera (-199.819, -119.816, 3.727), 31 ms old; screenshots of
the puppet and of overlay markers on known panels.

Not tested: a real network, random loss (only a deterministic "first send lost"), more than two players, a second puzzle or
any other gate type, a late join into a game that already has progress (the full-state resend exists but was not exercised in
the suite), loading a save, fullscreen, non-16:9 windows. The probes and test scripts are in witness-coop.

## How to play

Not playable together yet (prototype only, two copies on one PC). There is no installer or release; the dev setup is in the [witness-coop README](https://github.com/TommoDotCommo/witness-coop).

## Known issues

- Only panels and doors are synced. A panel that depends on another panel's power, cables, plates and lasers stay as in single-player.
- The puppet is a flat rectangle with a facing dot; it shows through walls (no depth test).
- Both players need the same world. Starting a new game on one side while the other has progress is not handled.
- Test hazard for the next agent: the game keeps saves in `%APPDATA%\The Witness` whatever
  folder the exe runs from. Starting a New Game from a copied install rewrote the owner's
  latest existing save and settings file there. Back up and hash that folder first.

## Next steps

1. Sync what a solve powers besides doors: cables and panels powered by panels (find the call, as for doors), plates, lasers, bridges. Check each with a natural solve and a state diff.
2. Real networking (Steam relay or direct IP), then random loss and lag runs with the chaos tool.
3. Late join and saves: exercise the full-state resend, define what happens when worlds differ.
4. Occlusion and a real puppet model instead of a flat rectangle.
5. Widen the suite to more puzzles and gate types and keep the QA level honest; add an installer and a release; read the Archipelago client for another view of the same hooks; finish the unchecked searches.
