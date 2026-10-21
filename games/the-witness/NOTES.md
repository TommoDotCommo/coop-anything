# The Witness: research notes

Status: research only, no code yet. Branch `game/the-witness`. Date: 2026-10-22.

## Game identity (tested build)

- Steam app 210970, build id 11370412 (Windows).
- `witness64_d3d11.exe`: PE32+ x64, 6864896 bytes, SHA-256 `8d672d444df6a6df7130f25a517bdab1af92731fe2138dc5b324d519561d55a5`
- `witness_d3d11.exe`: PE32 x86, 5657600 bytes, SHA-256 `6b8f8d9a3d595843335a96dc0d48fb133d0e77868c769673f7e4e6855811b8f6`
- Assets packed in `data-pc.zip` (about 2.7 GB). Ships `steam_api(64).dll`, `openvr_api.dll`, Bink 2, Ansel.
- Engine: `unknown` per `detect_engine` (133 files scanned, no signature match). Wikipedia calls it a proprietary engine.

## Observations (with sources)

- No co-op or multiplayer mod found. The game is single-player. Searched: this repo's directory, `games/`, `gh` issues and PRs (nothing), GitHub repository search ("the witness multiplayer", "witness coop", "witness64_d3d11"), and web search for Witness multiplayer/co-op mod. Not yet searched: Nexus, ModDB, archive.org, Reddit, speedrun/puzzle Discords, non-English communities. So "nothing found" is not "nothing exists".
- [jbzdarkid/witness-randomizer](https://github.com/jbzdarkid/witness-randomizer) (MIT, pushed 2026-01-09): external process, `Source/Memory.cpp` reads and writes the running game with typed `ReadData`/`WriteData` along an offset chain (globals, then an entity table indexed by entity id, then a field offset).
- [jbzdarkid/witness-trainer](https://github.com/jbzdarkid/witness-trainer) (MIT, pushed 2026-09-20): attaches with `OpenProcess`, finds addresses by byte-pattern signature scans (`AddSigScan`), re-scans when the game reloads. README features: noclip, sprint speed, FoV, disable saving, save/load position (X, Y, Z, theta), challenge timer. Says it should work with any Steam release; Windows only.
- Archipelago has a Witness world (`worlds/witness` in ArchipelagoMW/Archipelago); the client is a separate program run next to the game. How it hooks the game: not read yet.
- [ClementSparrow/TheWitnessExplorer](https://github.com/ClementSparrow/TheWitnessExplorer): Python tools to explore assets (no license, last push 2021).
- A 2016 Steam thread asked about screen-share "co-op" only; no mod ([discussion](https://steamcommunity.com/app/210970/discussions/0/458607699611570322/)).

## Memory layout work (2026-10-22, static, read-only on the exe)

Ran the randomizer's and trainer's signature scans over `witness64_d3d11.exe` (SHA-256 above) with a small script (`python -I`, parsing PE sections). Each signature matched exactly once.

| Item | Signature source | Result (RVA from module base) |
|---|---|---|
| GLOBALS | randomizer `Memory.cpp` `LoadPanelOffsets` | `0x62D0A0` |
| camera position | trainer `Trainer.cpp` (+0x10 adjust in trainer) | scan target `0x6304B0` |
| camera angles | trainer `Trainer.cpp` | `0x6303EC` |

Observed in the randomizer source: entities are read as `[GLOBALS + 0x18]` (pointer array), entry `id*8`, then a field offset. For `GLOBALS == 0x62D0A0` it hard-codes `POSITION 0x24`, `ORIENTATION 0x34`, `POWER 0x2A0`, `TARGET 0x2B4`. The same file also hard-codes a second build (`0x5B28C0`) and leaves `0x62B0A0` as TODO, so builds do differ.

Derived, not run: this Steam build matches a layout the randomizer already supports, so its entity field offsets should apply. Nothing was attached to a running game, so none of this is runtime-verified yet.

### Runtime probe (2026-10-22, first attach)

Read-only attach (`PROCESS_VM_READ`, Windows Python 3.11 + ctypes, no writes) to the copy at `local/witness-test`, module base `0x140000000`, game at its menu/start. Observed: camera position and angles read as sane floats; the entity array pointer resolved; entity `0x64` (Tutorial Straight) and `0x1B`, `0x1C` returned plausible positions (z 3.7 to 6.7) and `power` = `1.0, 1.0`. Not yet checked: that the camera values follow the player when moving, and that `power`/solved state changes when a panel is solved.

Side effect: launching the copy rewrote `Launcher.variables` in the real `%APPDATA%\The Witness` (a launcher settings file, not a save). The save folder was backed up first (14 files, hashes identical) to `local/saves-backup/`.

Second probe (after the owner started a New Game and went through the first door). Observed: camera position changed from (-36.1, -54.5, 4.4) at the menu to (-183.8, -163.4, 3.8) in the world, about 4.5 m from entity `0x64`, and stayed identical across two samples 5 s apart while the owner stood still. So the camera address tracks the player (moving-walk delta not yet sampled). Entity window `+0x290..+0x2D0`: solved/used panel `0x64` has `1.0` in many float slots (`+0x290`..`+0x2AC`) and `+0x2B4` = 216795; unsolved panel `0x65` has `+0x29C` = 1.0, `+0x2B4` = 110 and `+0x2BC` = 111. Unexplained: what 216795 means (not a plausible entity id), and which field is "solved". Needs a before/after sample of one panel being solved.

Third probe (before/after solving one panel). Correction: the panel at the player's feet was entity `0x182` ("Tutorial Bend", 0.6 m from the camera), not `0x65` (about 180 m away). Observed: solving `0x182` changed `+0x290`, `+0x294`, `+0x2A8`, `+0x2AC` to `1.0`, `+0x298` to int `1`, and `+0x2B0` to `1.322` (float; meaning unknown). After the solve its field pattern equals the earlier-solved panel `0x64`.

Static cross-check: the trainer's `solvedTargetOffset` signature (`84 C0 74 19 0F 2F B7`) resolves to `0x294` in this exe. The trainer reads `solved = (float@0x294 == 1.0)`, `hasEverBeenSolved = int@(0x294+0x04) = +0x298`, `state = int@(0x294-0x14) = +0x280`, name at `0x294-0x7C = +0x218`, traced edges at `0x294-0x6C = +0x228`. The runtime change at `+0x294` and `+0x298` matches that. So for build 11370412: solved flag = float at entity `+0x294`, ever-solved = int at `+0x298` (verified once, on one panel). `+0x280` (state) and the traced-edge data (needed to show the same path) were not sampled.

Camera: tracked entering the world (see second probe); smooth walking deltas not sampled.

Save-folder finding: the game keeps saves in `%APPDATA%\The Witness`, whatever folder the exe runs from, so a "test copy" does not isolate saves. Starting a New Game rewrote the latest existing `.witness_campaign` (and its png) there and added new files; the launch also rewrote `Launcher.variables`. Steam Cloud (`steam_autocloud.vdf`) may sync such changes (not checked). Back the folder up before any test launch; the owner chose not to restore here. Lesson for the playbook: check where a game writes saves before launching a copy.

## Derived (not tested)

- Player position and panel/entity state are reachable from outside the process, and the community already finds them by signature scan, so a mod can be an external or injected process without source.
- Sig scans tolerate patches better than fixed addresses, but this build's hashes above are the only ones tested.
- Co-op here means: show a second player (puppet), share which panels are solved and which doors are open, probably host-authoritative.

## Guesses

- The randomizer's panel-state offsets are likely reusable for syncing solved state.
- Puzzles are likely deterministic from the entity table, apart from the random-door and challenge sections.

## Unknowns

- Engine family and whether it has any networking code.
- Whether any script layer exists inside `data-pc.zip`.
- How a puppet player can be drawn (no spawn API known; may need an injected renderer or an overlay).
- How panel solving is represented in memory and in saves (randomizer `Panels.h` lists entity ids; `POWER`/`TARGET` fields look relevant), and how saves are written.
- Whether `witness64_d3d11.exe` has debug symbols or a known pdb.
- Whether the GOG/Epic builds differ from the Steam build.
- How the Archipelago client reads the game.

## Next

1. Runtime-verify the static results against a running copy: read camera position and one panel's `POWER`, compare with what the game shows.
2. Copy the install into `local/` and run in the copy only.
3. Finish research (Nexus, ModDB, Reddit, Discord) and ask the owner before any account use.
4. Decide hook style (external process vs injected DLL) with `coop-reverse-engineer`.
