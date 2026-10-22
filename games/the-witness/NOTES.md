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

## Entity census and puppet candidates (2026-10-22, read-only, build 11370412)

Launch note: the exe opens a graphics/video options dialog first, and nothing loads until it is confirmed. The command-line flag `-skip_config_dialog` (found in the exe strings, with `-windowed`, `-new_game`, `-no_hud`, `-non_interactive`, `-command`, `-width`, `-height`) skips it; launching `witness64_d3d11.exe -skip_config_dialog -windowed` reached the world with no user input. `powershell.exe` hung from this WSL setup, while Windows Python `subprocess.Popen` launched it fine.

The game's own classes carry no RTTI (only third-party library classes do), so entity types are told apart by the vtable pointer at the start of each entity (an address inside the exe's `.rdata`, `0x5156A8` to `0x521678` from the module base). Observed in the live world: 49,422 non-null entities of 48 types. Largest types: `0x518320` (26,675), `0x5156a8` (5,505), `0x516c88` (4,890), `0x51f2c8` (3,119), `0x517ed8` (2,067).

- Panels are type `0x518ca8` (594 instances). Entities `0x64`, `0x65` and `0x182` are all this type. (`0x65` is a panel too, about 180 m away from the start; the earlier "wrong panel" correction was about which one was at the player's feet.)
- A singleton of type `0x518090`, id `0x1e465`, sits at x,y equal to the camera's and z = 1.8, while the camera z is 3.51: 1.71 above it, i.e. an eye height over feet. Derived, not proven: this is the player entity. It would be the natural entity to read for the local player's position.
- Neighbouring entities of type `0x5183d0` (39 instances, ids from `0x1e467`) sit within a few metres of the player. Unknown what they are.

Not known: which of the 48 types draw a mesh, and whether moving an entity's `+0x24` position moves its rendering (the render transform may live elsewhere). That is the next experiment, and it needs a write to the test copy.

### Does moving an entity's position field move what is drawn? (2026-10-22)

Test on the live copy (the only writes made to the game so far; each restored and read back): wrote entity z (`+0x2C`) +3.0 m, held 10 to 24 s, then restored the original. Observed: the write took and stayed for the whole hold (the game did not overwrite it), the restore read back exactly the original. Entity `0x182` (panel at the player's feet, likely out of frame): owner saw nothing. Entity `0x64` (panel, about 8 m away): a screenshot taken during the displacement looks identical to the one before (same panel at the same pixels in the doorway ahead). Not proven: that the glowing panel in the screenshot is `0x64`.

Screenshot method that needs no installs: Windows Python `ctypes` (find the window with `EnumWindows`, `BitBlt` from the screen DC, write the PNG with `zlib`). Works on this windowed copy.

Derived: panel entities' `+0x24` position is gameplay data, not what places the mesh (the level geometry is likely baked or placed through another transform). So "reuse an existing panel as the puppet by writing its position" does not work. Other types were not tried (48 exist).

Imports of `witness64_d3d11.exe` that are plain files next to it, so they can be replaced with a proxy DLL without installing anything: `AnselSDK64.dll`, `amd_ags_x64.dll`, `openvr_api.dll`, `bink2w64.dll`, `steam_api64.dll`. It also imports `D3D11CreateDevice` (d3d11.dll) and `CreateDXGIFactory` / `CreateDXGIFactory1` (dxgi.dll), so a D3D11 overlay is possible by loading from a proxy and hooking `IDXGISwapChain::Present`. A MinGW-w64 C++ compiler is installed here; an overlay would also need the camera's view/projection (camera position and angles are known; field of view is not yet read).

## Proxy DLL: first code running inside the game (2026-10-22)

Result: a proxy for `AnselSDK64.dll` (MinGW-w64, 9 exports) loads into `witness64_d3d11.exe` in the test copy and logs from inside the process. Verified by: its log line (`proxy loaded: pid=... host=...witness64_d3d11.exe`), the log line `real Ansel DLL loaded` (the game called an Ansel export, so the stubs are exercised), memory climbing to 1.7 GB, and a screenshot of the normal rendered world. The original DLL is kept in the copy as `AnselSDK64_real.dll` (SHA-256 starts `3b0677ae1e01fc17`). The real install was not touched.

What went wrong first (useful to the next agent):
- Export forwarders in the `.def` file (`updateCamera = AnselSDK64_real.updateCamera`) made the loader report "AnselSDK64_real.dll was not found" although the file sat next to the exe. Fixed by making each export a naked stub that calls `LoadLibraryW` on the full path on first use and tail-jumps to the real function.
- That loader error is a Windows "System Error" box owned by `csrss.exe`, not by the game: the game process showed no window, 0 CPU and about 5.8 MB, which looked like a hang. The box stays on screen after the process is killed and shows up in later screenshots. Find it by enumerating top-level windows titled `<exe> - System Error` and close it with `WM_CLOSE`.
- Do no I/O in `DllMain` (loader lock): logging to a `\\wsl.localhost` path runs on a thread started from `DllMain`.
- A full-desktop screenshot (pid 0 in my capture script) is the quick way to see such boxes.

Imports of the exe from `AnselSDK64.dll` were not listed by my first `objdump` filter; all 9 exports are stubbed. Per-frame use of `updateCamera` is derived from the name, not observed.

## Overlay: drawing inside the game's frame (2026-10-22)

Result: from the proxy DLL, a hook on `IDXGISwapChain::Present` draws screen-space markers at world positions, and they land on the right objects. Screenshot evidence: a magenta square on the golden panel in the doorway ahead (entity `0x293`) and one on the stone column (entity `0xc335`), both within a few pixels of where the model predicts (`0x293`: predicted (1055, 689) of 2405x1353, observed on the panel).

How (source in `local/the-witness-mod/`, not published yet):
- Hook: create a dummy D3D11 device and swap chain in the proxy, read `Present` from its vtable (index 8) and overwrite that slot (`VirtualProtect`), since swap chains of the same implementation share the vtable. The game's `Present` then calls our function, which draws and calls the original. The game ran normally afterwards (log: `Present hook called`, resources created, backbuffer 2405x1353, format 28 = `R8G8B8A8_UNORM_SRGB`).
- Drawing: runtime-compiled HLSL (`d3dcompiler_47.dll` via `LoadLibrary`), one instanced quad draw with a constant buffer of rectangles, depth and blend off.
- Camera model (read in-process from module base + `0x6304C0` position, `0x6303EC` angles): `forward = (cos(phi)cos(theta), cos(phi)sin(theta), sin(phi))`, `right = (sin(theta), -cos(theta), 0)`, `up = (-sin(phi)cos(theta), -sin(phi)sin(theta), cos(phi))`, z up. Field of view: vertical 60 degrees fits (about 89 degrees horizontal at 16:9); calibrated from two points in one frame, so treat as +-2 degrees.

Not verified yet: this was calibrated on the title-screen camera (pitch exactly 0). The same addresses followed the player earlier, but the projection has not been checked while walking in the world. `updateCamera` (Ansel) was never called in about 100 s of running, so it does not give a per-frame camera; the camera comes from game memory.

Walking check (same day). Method: launch the copy with `-skip_config_dialog -windowed -new_game` (starts straight in the world), markers on the 16 nearest in-front panels from the randomizer's id list, keypress sent with `SendInput` (scancode `W`, 1.5 s) after bringing the window to the foreground, screenshots before and after. Observed: the camera went from (-201.4, -114.8, 3.73) to (-199.9, -119.5, 3.73) at heading theta = -1.257; the model's forward vector (cos, sin) = (0.306, -0.952) times 4.9 m gives (+1.5, -4.7), the observed delta. A marker sat on the glowing end of the start tunnel before and after the walk, and the far markers spread outward as expected. Two different headings (theta 0.225 on the title screen, -1.257 in the tunnel) both fit, so the angle convention is right in both. Not tested: turning in place with the mouse while watching markers (the math was checked at two headings only), a non-16:9 window, and fullscreen. The overlay has no depth test, so markers show through walls.

## Two players on one PC: pose exchange and puppet (2026-10-22)

Result: two copies of the game, each with the proxy DLL, exchange camera pose over UDP on localhost and each draws the other as a puppet through the `Present` overlay. Steam allowed two game processes of the same app from two folders at once.

Setup: second install copy `local/witness-test-2` (3.9 GB, about 33 s to copy). Per-copy `witness_coop.cfg` (`id`, `listen`, `peer` ports 47001/47002). A thread in the proxy sends `{magic, id, seq, pos[3], angles[2]}` (36 bytes) at 30 Hz and keeps the latest packet from the peer. The overlay draws the remote player as a tall green rectangle from feet (eye minus 1.71 m) to head, plus a yellow dot 0.7 m ahead of the head along the remote heading. Both games were started with `-skip_config_dialog -windowed -new_game` and placed side by side with `SetWindowPos`.

Observed, with a before/after test: A (id 1) was sent a `W` keypress for 1.6 s and walked about 5 m. A's camera read from its memory: (-199.819, -119.816, 3.727). B's log: `recv ... from id 1 ... pos (-199.82,-119.82,3.73)` and `overlay: remote id 1 age 31ms`. An earlier packet (`recv #600`) showed an intermediate position (-200.59, -117.45), so positions update during the walk. B's screenshot shows the green body ahead in the start tunnel at the expected size and place. A drew nothing for B (B is behind A, so the puppet is behind the camera).

Not tested: more than two players, a real network (LAN or internet; localhost only), packet loss or reordering (no sequence check yet), the puppet while the remote player turns or jumps, a puppet that is occluded by walls (no depth test), or the puppet drawn while the local player has the panel-solving cursor up. The body is a flat rectangle, not a model. Pose is the camera (eye) position, so crouch or prone states are not separated.

## Shared panel and door state (2026-10-22 to 10-24)

Goal: when one player solves a panel, the other player's game shows it solved and the gated door opens. Design follows [effect-capture-story-sync](../../playbook/effect-capture-story-sync.md): the machine where it happens detects it, sends numbered effects, the other side applies them directly and never replays the player's input.

### What the game does on a solve (observed, build 11370412)

Entity type names come from the game's own type descriptors: `entity+0x08` points to a descriptor whose `+0x08` is the name (the trainer reads names the same way, and the door function compares `[entity+8]` with a global descriptor). 48 types in the live world, among them `Machine_Panel` (594), `Door` (327), `Power_Cable` (268), `Pressure_Plate` (263), `Multipanel` (41), `Group` (741), `Inanimate` (26,675), `Collision_Volume` (4,890), `Human` (1, the player, entity `0x1E465`). Descriptors in the exe: `Machine_Panel` `0x62E670`, `Door` `0x62DE70` (RVAs).

A natural solve of the first panel (`0x64`), diffed over every entity between a snapshot before, an idle snapshot (noise baseline) and one 4 s after (`wsnap`/`wdiff2` scripts, 49,414 entities, bytes `+0x20..+0x400`): 15 entities changed in a way idle time does not explain.
- Panel `0x64`: `+0x278` `0 -> 1.0`, `+0x280` `0 -> 1` (state "Solved"), `+0x290` `0 -> 1.0`, `+0x294` `0 -> 1.0` (solved flag), `+0x298` `0 -> 1` (ever solved). `+0x2B0` ticks all the time (a clock), so it is not part of a solve.
- The panel's target id at `+0x2B4` is `0x34EDB` (an `Inanimate` group node). The `Door` entity `0x34EDA` got three power words `0 -> 1.0` (`+0x1D0`, `+0x1D4`, `+0x1E4`), an `Inanimate` `0xD05D` the same, and the door's panel and four collision/mesh children moved (the door swings open).
- Panels power 18 different target types (doors, other panels, cables, groups, plates, ...), and 341 of 594 panels have no valid target, so a solve cannot be mirrored by one special case.

### Writing fields does not open a door

Writing exactly those 11 words into the other game (all writes succeeded) changed nothing in the world: the door stayed closed. The door's opening is started by a function call, the one the trainer calls (`_doorOpenStart(door, 1.0)`). In this build it is `base + 0x8E4B0` (found by the trainer's signature `F2 0F 10 41 24 48 89 68 10`, start 15 bytes before the hit; disassembly shows `this` in `rcx`, the amount in `xmm1`, a type check against the Door descriptor and recursion over linked entities). Calling `DoorOpenStart(0x34EDA, 1.0)` inside the other game moved panel `0x64` to exactly the position the real solve produced there, `(-186.232, -158.897)`, with no crash.

A door is "open" when its power word at `+0x1D0` equals `1.0` (10 doors read as open by default in a fresh game, identical in both games, plus `0x34EDA` after the call).

### Implementation (in the proxy DLL; source not published yet)

- Detector thread (10 Hz): every `Machine_Panel` solved (`+0x294 == 1.0` and `+0x298 == 1`) and every `Door` open (`+0x1D0 == 1.0`) is compared with its last known value; a change becomes an event.
- Transport: events are numbered per stream epoch, acknowledged cumulatively, resent every 150 ms until acknowledged, applied once and in order, duplicates dropped. A peer restart (new session id in its pose packets) starts a new epoch and a full resend of every solved panel and open door; applying is idempotent.
- Applier (render thread, between frames): a remote panel is written as solved (the five words above); a remote door is opened by calling the game function, after checking the entity's type. Both mark the entity as suppressed for 5 s so the local detector does not send the change back.
- Control channel for development: `witness_cmd.txt` (`door <hexid>`), polled by a thread, run on the render thread.

### Two bugs found by running it (both are in the playbook's warnings)

1. A baseline taken while the world was still loading saw 39 transient "open" doors; the other game applied 30 of them. Fix: nothing is detected, resent or applied until the camera is valid and the panel and door counts are identical on three scans 3 s apart.
2. The echo guard was only a "known" flag, which the detector reset when the door was not yet powered a moment after the call, so 24 doors bounced back. Fix: a per-entity suppression window (5 s) after every remote apply.

### Test

A bot (Windows Python, `SendInput`) plays player A: walks 46 m to the first panel (3 s key presses with a distance check), aims by feedback from the camera angles (0.00255 rad per mouse count horizontally, 0.00237 vertically), clicks to enter the puzzle, moves the cursor onto the start circle (0.218 m left of the panel centre, 3.6 mouse counts per pixel in puzzle mode), presses, drags to the end and clicks to finish. First run, no flags: A logged `local panel 0x64 solved -> event` and `local door 0x34eda opened -> event` (11 events sent in total, 11 acknowledged); B logged `remote panel 0x64 applied as solved` and `remote door 0x34eda opened (call ok)` and emitted nothing back; a state diff over all 594 panels and 327 doors read from both games' memory was identical (1 solved panel, the same 10 open doors).

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
- How a puppet player can be drawn: moving a panel entity's position does not move it on screen; a D3D11 overlay from the proxy DLL works (see below); other entity types were not tried.
- How panel solving is represented in memory and in saves (randomizer `Panels.h` lists entity ids; `POWER`/`TARGET` fields look relevant), and how saves are written.
- Whether `witness64_d3d11.exe` has debug symbols or a known pdb.
- Whether the GOG/Epic builds differ from the Steam build.
- How the Archipelago client reads the game.

## Next

1. Runtime-verify the static results against a running copy: read camera position and one panel's `POWER`, compare with what the game shows.
2. Copy the install into `local/` and run in the copy only.
3. Finish research (Nexus, ModDB, Reddit, Discord) and ask the owner before any account use.
4. Decide hook style (external process vs injected DLL) with `coop-reverse-engineer`.
