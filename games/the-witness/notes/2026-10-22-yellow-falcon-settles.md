# 2026-10-22: retain acknowledged events until game application succeeds

Implementation: witness-coop commit `bfe1b7e` (`yellow-falcon-settles`).

## Observations

- `SyncCore::onPacket` acknowledges ordered events when queued, before the render thread applies them. Previously `applyRemote` removed events even when the entity was absent or `applyDoorOpen` returned false; the latter also incremented applied counters and installed an echo guard for a refused call.
- Added a deterministic, game-free receiver regression (`proxy/core/tests/apply_retry.cpp` in witness-coop). Before the fix, missing-panel, missing-door and refused-door cases failed; wrong-type rejection passed.
- After the fix, all four cases pass. Each retry case catches a `noApplyRetry` twin. They check ACK-before-application, convergence without another packet, unrelated-door progress, accurate applied counters, and no local echo after success.
- `make -C proxy/core test`: deterministic regressions pass, 500 clean fuzz sessions pass, all 10 existing fuzz twins caught. `make -C proxy`: DLL builds. `git diff --check`: clean.

## Derived

- Transport reliability alone does not guarantee state application: the receiver must retain responsibility for an acknowledged event until its game adapter succeeds.
- Retries make at most one attempt per queued event per frame, within the existing 1,024-event receive bound. These panel/door changes are monotonic and independent, so a temporarily unavailable entity need not block another entity.

## Unknowns

- No real-game missing entity or refused call has been induced; these failure modes are simulated at T0.
- First real-game baseline attempt failed during startup: A never logged the overlay patch or networking startup, bot entity lookup returned an unreadable table, and measured 0 fps. B reached the stable-world gate. Save hashes, redirect and cursor checks passed; the suite released its slots. A second fresh baseline attempt passed: one solved panel and ten open doors identical in both games, no echo from B, no resends, 30.0 fps, save hashes/redirect/cursor checks passed, slots released. The first startup failure did not recur; its cause remains unknown.

## Follow-up: real-game restart catch-up

### Observations

- Reviewed the hub's world-state build order (`playbook/how-coop-mods-work.md`, stage 4), effect capture and desync guidance. Chose the missing real-game join/rejoin check: full-state resend already exists, but only the game-free simulator covered it.
- Added selective `tools/restart.sh B` / `tools/install.sh B`; the surviving A process, config and loaded DLL are preserved. Newly launched processes are registered immediately rather than after both launches finish.
- Added clean and disabled-catch-up mutant scenarios. Preconditions read both worlds from memory and require the original solve to have reached B and all sender events to have been acknowledged. After B restarts with `-new_game`, A must retain its PID and state and B must have a new PID and complete census. The clean case requires the whole solved/open state to match and no B echo; the mutant specifically requires the first panel and its door to be missing on B.
- `bash -n tools/restart.sh tools/install.sh`, Python compilation, DLL build, and T0 (application regressions, 500 clean fuzz sessions, all ten fuzz twins) pass. Real-game `tools/qa.sh 'restart catch-up'`: both scenarios PASS. Clean restored one solved panel and ten open doors, with no echo. Mutant left exactly panel `0x64` and door `0x34eda` missing on B, after the original two events had been acknowledged. Both runs kept real save hashes unchanged, redirected instance saves, left the cursor free and measured A at 30.0 fps. All leased games were released. Implementation: witness-coop `df7b276` on `yellow-falcon-settles`.

### Derived

- Restarting a receiver only after the original events are acknowledged distinguishes snapshot catch-up from retransmission of outstanding deltas.

### Unknowns

- Real-game first-time late join, same-process save reload and WAN behavior remain untested.

## Follow-up: second tutorial puzzle exposes skipped door zero

### Observations

- Read Tutorial Bend `0x182` using the panel graph offsets from [witness-randomizer Memory.cpp](https://github.com/jbzdarkid/witness-randomizer/blob/master/Source/Memory.cpp), build branch `GLOBALS == 0x62D0A0`: positions at `+0x3C0`, flags `+0x3C8`, connections `+0x3D0/+0x3D8`, counts `+0x3B4/+0x3B8`. Three nodes: `(0.8,0.25)`, `(0.8,0.75)`, `(0.2,0.75)`, connections `0-1`, `1-2`.
- Standing camera command `tp -182.71 -164.90 3.727 0.303 -0.066` sees the puzzle screen head-on. The actual eye settles at z about 3.896 on the floor. A proxy back-buffer shot shows the start circle at `(1016,640)` in 1920x1080; entering aiming mode initially puts the cursor at `(960,540)`.
- Free cursor placement uses 112 counts right and 200 down at this viewport. A short constrained trace stopped before the end; 40 frames of `(0,-18)` then 40 of `(-18,0)` reaches both segment ends. Clicking finishes the natural solve.
- Memory after the solve: A and B both contain solved panels `0x64` and `0x182`; A has 11 open doors, B 10. The only difference is door ID `0x0`, open only on A. `tools/state_diff.py` exited 1 with exactly that difference.
- Root cause in `proxy/state.cpp`: the entity scan starts at ID 1. The read-only census starts at 0 and finds 327 doors, while the adapter sees 326. `GameEntity(0)`, packet validation and the door-call bridge otherwise accept zero.
- Added `wbot2.py`, shared `wcontrol.py`, read-only `wpanel.py`, and a second-tutorial suite scenario that requires both panels solved plus both specific doors open. A skip-zero mutant must retain the second panel solve on B but lose door zero. The fresh pre-fix clean scenario FAILed with exactly door zero missing on B, while both bots reported solved from memory. The mutant also reproduced that same missing-door state. The scanner fix includes table index zero by default, and `skip_entity_zero=1` restores the old scan for the mutant. Post-fix clean/mutant state checks both PASS: clean has 2 panels and 11 doors on both sides, no B echo; mutant has both panels but lacks only door zero on B. Save hashes, redirect, free cursor and 30.0 fps checks pass; no games/leases remain. The wrapper unexpectedly returned 1 after ALL OK; a repeat with scripts left unchanged returned 0, with both scenarios PASS and all resources released. The earlier status mismatch did not recur; cause unknown. Implementation: witness-coop `cfb67a0` on `yellow-falcon-settles`. Shell syntax, Python compilation and whitespace checks pass. DLL build and all existing T0 tests pass.

### Derived

- Table index zero is valid in this build; zero cannot be used as a table-index sentinel during census. The mod/probes call these normalized table indices entity IDs; the game's stored engine IDs use the separate base described below.
- Correction to the earlier target-type interpretation: this repo's event IDs and probes use entity **table indices**, while the game's target fields hold engine IDs. Read `table_base_id = u32([base+0x62D0A0]+8)`, observed `1`. At `exe+0xB9F00`, the game reads panel `+0x2B4`, subtracts this base ID, indexes `[globals+0x18]`, checks the Door descriptor, and calls `exe+0x8E4B0`. Thus target `0x34EDB` means table index `0x34EDA`, and target `1` means table index zero. The earlier notes' claim that the first target is an Inanimate at index `0x34EDB` was a namespace error. `wpanel.py` now reports both raw target and normalized table index.

### Unknowns

- Cables and powered-panel replication remain unimplemented. The second tutorial panel is another door gate, so it first exposed this earlier correctness gap.

## Follow-up: panel power-on progression

### Observations

- A read-only target census found a simple already-powered tutorial panel at table index `0x18AF`, raw target `0x1C` = target table index `0x1B` (Machine_Panel, initially requested and visible power both zero). Its 7-node graph has a middle horizontal route that separates the black and white squares.
- Natural proxy input solve: teleport above the standing point `(-158.745,-69.925,10.0)`, yaw `2.749`, pitch `0.003`; the eye settles at z `6.7143`. Enter aiming, 80 counts left to the start, click, 40 frames of +18 x, click finish. A source solve lights up target `0x1B` on A; B gets the solved source but leaves its target at `(0,0)` power. In-game shots show the second panel lighting on A. Owner save hashes unchanged across the research session; slots released.
- A first long teleport at ground height was displaced by collision resolution. Arriving above the ground works; the new bot checks the actual camera position and retries up to three times before failing.
- Disassembly of build 11370412 at `exe+0xC2890` takes a panel pointer, sets requested power `+0x2A4` to `1.0`, handles redraw/randomization and linked doors. Visible power `+0x2A0` follows in the normal game loop. The applier type-checks Machine_Panel and checks the function prologue, then calls this on the render thread.
- Added deterministic power-on core tests; both live sync and restart recovery failed before the new event. After implementation they pass, including dropped first sends, refused calls held past the echo interval, no echo, and ensuring a powered target is not marked solved. Disabled-power, no-apply-retry, no-echo-guard and no-resync twins are caught. Existing application regressions plus 500 clean fuzz sessions and all ten old fuzz twins pass. DLL build passes.
- New event kind 3 shares panel power-on via the reliable queue and full-state snapshots. Power state is part of the stable-world signature and has its own echo guard. The wire magics are v2 (`WTP2`, `WTE2`, `WTA2`) to avoid an older receiver silently skipping the new vocabulary. Both copies must use the updated DLL.
- Real-game `tools/qa.sh 'panel power chain'`: clean and disabled-power mutant both PASS, command exit 0. Clean also restarts B after all original events are acknowledged: both worlds match 2 solved panels, 10 open doors and 238 powered panels. The target remains unsolved, B emits no power echo and applies one power event. The mutant preserves both solved panels and doors but has 237 powered panels on B, missing only `0x1B`. Save hashes, instance save redirect, free cursor and 30.0 fps checks pass; own games and slots released.
- Protocol compatibility fixture: checksummed v1 event and pose are ignored, while matching v2 controls are accepted. All power tests and the existing 500-session fuzz/twin suite pass. Implementation: witness-coop `9799af8` on `yellow-falcon-settles`. Python compilation and `git diff --check` pass; final DLL rebuild passes.

### Derived

- Reading requested power for detection catches the discrete activation while letting each engine animate its visible brightness. The real-game assertion requires both requested and visible power to reach 1.

### Unknowns

- This adds power-on progression only, matching the existing solved/open progression model. Power-off switching, conflict/authority rules for reversible state, and cable sync remain unimplemented.
