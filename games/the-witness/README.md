# The Witness (2016)

| | |
|---|---|
| Engine | In-house engine (Thekla). Family not confirmed: `detect_engine` matched no signature |
| Versions tested | Steam build `11370412` (app `210970`), `witness64_d3d11.exe` SHA-256 `8d672d444df6a6df7130f25a517bdab1af92731fe2138dc5b324d519561d55a5`, Windows |
| Projects | none found. Related tooling: [witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) (MIT), [witness-trainer](https://github.com/jbzdarkid/witness-trainer) (MIT) |
| QA level | none |
| Last updated | 2026-10-22 |

## Where things stand

There is no co-op for The Witness, and no mod was found; this is research only, with no
mod code yet. The groundwork is better than it looks: two MIT tools already attach to
the running game from outside and read and write player and puzzle state, and on this
Steam build the camera position, the entity table and the panel solved flag were read
from a live copy. The open problem is showing a second player: nothing found draws one.

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

Not known: how to draw a second player, how to safely write a solved panel and open its
door or cable, how saves are written, and whether other store builds differ (the
randomizer already knows two builds with different offsets).

## What we changed

Nothing yet. This round was research and read-only probing; no game file was modified.

## QA

No automated tests. Everything above was checked by hand on one machine: a read-only
attach with `PROCESS_VM_READ` from Windows Python, before and after solving one panel.
The probe scripts are not published yet.

## How to play

Not playable together yet.

## Known issues

Test hazard for the next agent: the game keeps saves in `%APPDATA%\The Witness` whatever
folder the exe runs from. Starting a New Game from a copied install rewrote the owner's
latest existing save and settings file there. Back up and hash that folder first.

## Next steps

1. Decide how a second player is drawn (overlay, injected renderer, or reusing a game entity) with the reverse-engineering skill.
2. Work out a safe write path for solved state, and sample the state field and traced-edge data (`+0x280`, `+0x228`).
3. Read the Archipelago client for another view of the same hooks.
4. Finish the unchecked searches, then build a bot that drives a panel solve and a state diff.
