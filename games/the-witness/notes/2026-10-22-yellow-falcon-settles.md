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
