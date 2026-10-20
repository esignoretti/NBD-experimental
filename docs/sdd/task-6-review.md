# Task 6 Review — TRIM / WRITE_ZEROES / read-only (7473656d, base eabfa63c)

Reviewer: agent `reviewer`, model `cubbit/vllm/mimir`, run `7c72db34-66c9-48c9-80fe-0b3a9a376bc4`.
Persisted by the controller: the `reviewer` agent has no write tool, so it returned
the review inline and the controller wrote this file. The text below is the
reviewer's, verbatim.

## Spec Compliance

| # | Brief requirement | Verdict | Evidence |
|---|---|---|---|
| 1 | Files touched: `internal/adapter/nbd/handlers_write.go`, `connection.go` + tests | ✅ | diff stat: 5 files (+687/−16) |
| 2 | `handleTrim` / `handleWriteZeroes` exist with the specified signatures | ✅ | `handlers_write.go:137-139`, `:156-158` (both delegate to `handleDiscard`; signatures preserved) |
| 3 | Check order 1 flags → 2 ReadOnly → 3 Length==0 → 4 overflow (EIO) → 5/6 range → store → FUA → reply | ✅ | `handlers_write.go:174-175`, `:181-190`, `:189-194`, `:199-201`, `:212-217`, `:219-223`, `:243-262` |
| 4 | Flag masks: TRIM = FUA, WRITE_ZEROES = FUA\|NO_HOLE | ✅ | `:138` (`FlagFUA`), `:157` (`FlagFUA\|FlagNoHole`); `handleWrite` uses `FlagFUA` only (`:33`) |
| 5 | `decision:` markers at the mandated sites | ✅ | `:129` (TRIM over-delivery), `:143` (punches not zero-buffer WriteAt), `:209` (per-command out-of-range code), `:274` (reply-before-drain) |
| 6 | Idle arm inside `if !h.draining()`, with the mandated comment + `decision:` note | ✅ | `connection.go:205-225` (comment `:206-218`, budget `:219-224`) |
| 7 | Arm `Read` before the WRITE payload read | ✅ code / ❌ test | arm present `handlers_write.go:70`; the plan-mandated test is **absent** (see Issue 1) |
| 8 | Integration tests; no `DataExtents` assertion; `DrainAllUploads` after Flush | ✅ | `integration_test.go:487-522`, `:523-556`; pre-punch non-empty assert at `:471` |
| 9 | Step 9 command run | ⚠️ Cannot verify from diff | report-only (`go build ./... && go test -race …` → ok nbd/pkg/common); not re-run per instructions |
| 10 | All 15 exact test names | ✅ | all 15 present: `connection_test.go:1176,1192,1234,1254,1273,1292,1309,1336,1363,1398,1413,1436,1465`; `integration_test.go:487,523` |
| 11 | Read-only fixture as a second helper, not a mutation | ✅ | `readOnlyExport` at `connection_test.go:147-155` |
| 12 | `mapError` on every error path; `writeReply(cookie, mapError(err), nil)` | ✅ | `:246`, `:249`, `:259`; rejection paths pass literal `NBDError` values, which is correct |
| 13 | Zero-length = uniform `EINVAL`, no drain, all four commands | ✅ | READ `handlers_read.go:37`, WRITE `handlers_write.go:44`, TRIM/WRITE_ZEROES `:189-194` (comment: no bytes in flight) |

## Priority checks (the 10 the task named)

1. **Zero-length uniform `EINVAL`, no drain** — ✅. READ/WRITE/TRIM/WRITE_ZEROES all `ErrEINVAL`; the discard path returns `writeReply` directly, never `rejectWrite`, so no `io.CopyN` runs (`handlers_write.go:189-194`). `TestConnection_TrimZeroLengthIsEINVAL`/`WriteZeroesZeroLengthIsEINVAL` prove the stream stays synchronized with a following READ.
2. **Metadata punch cannot be skipped** — ✅. `h.meta.PunchHole` then `bs.PunchHole(ctx, string(res.PayloadID), res.PreOpBlocks, …)` (`:243-250`); the comment correctly cites `pkg/block/engine/readwrite.go:308`'s `len(currentBlocks) > 0` guard and `pkg/metadata/sparse.go:80` as the sole authorization check.
3. **Out of range refused, never clamped; subtraction guard** — ✅. `req.Offset > math.MaxUint64-uint64(req.Length)` → `ErrEIO` (`:199-201`); two range checks, no clamp anywhere; `TrimOffsetPastEndIsEINVAL` covers the underflow case.
4. **Different out-of-range codes** — ✅. `handleTrim` passes `ErrEINVAL`, `handleWriteZeroes` passes `ErrENOSPC`; both feed `outOfRange` at `:212-223`.
5. **WRITE_ZEROES is not a zero-buffer `WriteAt`** — ✅. Test asserts both halves: `st.punches != 1` and `st.writes != 0` (`connection_test.go:1381-1386`), and the fixture pre-populates `st.data` so `writes == 0` is meaningful (`:1373`).
6. **NO_HOLE accepted on WRITE_ZEROES, rejected on WRITE** — ✅. `:157` vs `:33`; `TestConnection_WriteZeroesWithNoHoleIsAccepted` and `TestConnection_WriteWithNoHoleFlagIsEINVAL`.
7. **Idle/Read split, two halves** — half-pinned. Idle arm is pinned by `TestConnection_IdleDeadlineClosesASilentConnection`; the payload `Read` arm has **no** test (Issue 1).
8. **Read-only → EPERM while still advertising** — ✅. `handlers_write.go:181-190` with the handshake reasoning; `TestConnection_TrimIsEPERMOnReadOnlyExport` / `WriteZeroesOnReadOnlyExportIsEPERM` assert `punches == 0`.
9. **FUA on TRIM/WRITE_ZEROES commits before reply** — ✅ code, untested. `:252-262` mirrors WRITE; no test exercises FUA on either new command (Issue 3).
10. **`mapError` everywhere, returns `NBDError`** — ✅. All store/context failures route through `mapError`; rejections use literal `NBDError` constants, which is the same type.

## Strengths

- The metadata-punch comment block (`handlers_write.go:225-242`) is unusually good: it states *why* the punch is mandatory (authorization, not just reclaim) and names the exact engine line that makes a `nil` `PreOpBlocks` silently skip the reap. That is the one thing a future refactor would most plausibly break.
- No clamp anywhere, and the two-check range refusal is justified against the underflow a clamp would produce (`:203-217`). This is the spec's §4.4/§4.6 intent, not the earlier draft's.
- `TestConnection_WriteZeroesThenReadIsZeros` pre-populates the store so `st.writes == 0` is falsifiable — the report's Step-8 bite proof (`the store saw 0 punches, want 1` / `1 writes, want 0`) confirms both halves bite independently.
- `readOnlyExport` is a genuine second constructor, not a mutation of the shared `testExport` return (`connection_test.go:147-155`), exactly as the brief required.
- Integration tests assert the manifest, not `DataExtents`, and call `DrainAllUploads` after Flush with a load-bearing pre-assertion (`integration_test.go:471`) — the brief's "one more trap" is handled.
- Both rewritten doc comments match the code: `connection.go:20-33` and `pkg/adapter/nbd/config.go`'s `Read`/`Idle` docs describe the arm that actually runs (`config.go` Read: "bounds reading its payload"; Idle: "bounds the wait for a request header … 0 means no idle timeout").

## Issues

**Important — the plan-mandated payload-`Read` test is missing; nothing pins that half of the split.**
`TestConnection_PayloadReadIsBoundedByRead` is required by the plan (plan `docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md:2463`, and repeated as an unchecked verification item at plan:2873) but exists nowhere in the repo (`grep` finds it only in the plan). The plan states the measurement that makes it necessary: deleting the arm at `handlers_write.go:70` leaves the whole package green, because `TestConnection_WriteTruncatedPayloadIsAnError` uses `testTimeouts()` (Read 5s == Idle 5s) **and closes the client**, so the read ends on EOF and no deadline is consulted (`connection_test.go:991-993`). I confirmed the corollary: no `Timeouts` literal in the package has `Read < Idle` — the four are `5s/5s/5s` (`:172`), `5s/5s/200ms` (`:723`), `1s/5s/5s` (`:953`), `5s/5s/200ms` (`:1468`) — so no existing test can distinguish the two budgets on the payload path.
Why it matters: the arm is a real behavioral contract (`Read` bounds finishing a request already started) that a future refactor could delete with a green suite — precisely the failure the brief's own Step 5 text describes as the reason this task exists.
Smallest fix — add the test the plan specifies (and this also closes the "judge independently" question: it is **Important**, not acceptable-minor, because the plan both mandated it and predicted the green-suite outcome):
```go
func TestConnection_PayloadReadIsBoundedByRead(t *testing.T) {
    // Timeouts{Read: 200ms, Idle: 5s}: the inverse of the header test.
    // Complete the handshake, send a WRITE header declaring 4096 bytes,
    // send 100 bytes of payload, then go silent WITHOUT closing. Only the
    // deadline can end this, and it must be Read's. Assert serve returns
    // non-nil within ~1s and that st.writes == 0.
}
```
Note the framing: the implementer followed the brief faithfully — the brief's Step 5 names only `TestConnection_IdleDeadlineClosesASilentConnection` and its 15-test list omits this one. This is a **brief-vs-plan gap**, not implementer negligence; the controller should either add the test or adjudicate the discrepancy. I am reporting it rather than dropping it because the plan is explicit and its own checklist item is unmet.

**Minor — FUA on TRIM/WRITE_ZEROES is implemented but untested.** `handlers_write.go:252-262` runs `h.commit` before the reply, matching WRITE, but no test sends `FlagFUA` on either new command (the only FUA tests are READ `connection_test.go:597` and WRITE `:1038`). Check 9 rests on reading the code. A cheap test would send TRIM+FUA against a store whose `Flush` records a call and assert `st.flushes == 1` and the reply follows. Plan-level omission (the brief's test list has no FUA-on-discard case).

**Minor — `res.PayloadID` is used unguarded, diverging from NFS DEALLOCATE.** `handlers_write.go:248` passes `string(res.PayloadID)` with no emptiness check, where NFS guards with `if res.PayloadID != "" && length > 0 && offset < res.File.Size` (`internal/adapter/nfs/v4/handlers/deallocate.go:81`) and clamps via `punchLen`. On NBD's normal path this is unreachable: `Export.PayloadID` is resolved once at handshake from a regular file (`volume.go:147`) whose `buildPayloadID` never returns empty (`pkg/metadata/file_helpers.go:23`), and `sparse.go:76` copies `file.PayloadID`. But an empty `PayloadID` **is** reachable if the volume file loses its last link or is clobbered by a rename while the connection is open (`file_remove.go:220`, `file_modify.go:1594`, `file_recycle.go:143` all set it to `""` on a surviving file), in which case `bs.PunchHole` would be called with `""` and reap/write against a non-existent payload rather than refusing. Low likelihood (NBD has no unlink; another protocol would have to race it), no crash. The implementer flagged this honestly; it is a **plan-level omission** — the brief's snippet itself omits the guard. Worth a one-line `if res.PayloadID != ""` guard for parity, or an explicit note that NBD deliberately omits it.

**Minor — `bs.PunchHole` is called with the bare `ctx`, unlike every sibling store call.** `handlers_write.go:248` passes `ctx`; `handleWrite` wraps with `common.WithRequestDeadline` (`:103`) and `handleRead` likewise (`handlers_read.go:78`). A full-device `fstrim` on a large export runs the reap plus the 1 MiB zero-overwrite loop (`pkg/block/engine/readwrite.go:325-340`) with no deadline. The report discloses this and notes the brief's snippet says `ctx`, so it is plan-level; but the divergence from the other handlers is real. Confirm the omission is intended, or wrap the punch the same way.

**Minor — stale test failure message.** `connection_test.go:738` still reads `"serve did not return within 2s of a 200ms read timeout"` after the fixture was inverted to `Idle: 200ms`. The assertion is right; the message now names the wrong timer, which is exactly the confusion the fixture change was meant to remove.

**Minor — `punchErr` is dead test surface.** `storeFake.punchErr` is declared (`connection_test.go:37`) and honored (`:109-110`), but no test in the package sets it (only `readErr` is ever set). The engine-punch failure arm of `handleDiscard` (`handlers_write.go:248-250`) is therefore unexercised. Either add a `st.punchErr = errors.New("boom")` case asserting `NBD_EIO`, or drop the field.

## Also assessed

- **`TestConnection_PartialHeaderTimesOut` inversion is correct, not a weakened pin.** The committed fixture (`Read 200ms / Idle 5s`) passed only because the old loop armed the header wait from `Read` unconditionally; after the fix that wait uses `Idle`, so the old fixture would take 5s and fail its 2s bound. Inverting to `Read 5s / Idle 200ms` (`connection_test.go:720-741`) re-points the pin at the timer that now governs a partial header, and a silent half-header genuinely belongs to `Idle` — the same shape NFS uses (`pkg/adapter/nfs/connection.go` `resetIdleTimeout` guarded on `Idle > 0`, header read via `Read`). The pin still tests something real (a stalled client must not hold the connection open); it just names the right budget. The brief itself documents the old fixture as unable to catch the arm.
- **Doc corrections match the code.** `connection.go:20-33` and `pkg/adapter/nbd/config.go` `Read`/`Idle` describe the split as implemented. Confirmed both.
- **`PayloadID` reachability** — see Minor above; not reachable on NBD's happy path, reachable via a cross-protocol unlink race, no crash.
- **`NO_HOLE` semantics (observation, possibly plan-level).** The code accepts NO_HOLE but still punches (frees CAS chunks and zero-overwrites) rather than genuinely allocating the range as the flag's literal intent asks. The design spec (§4.3, `…design.md:304-307`) and the brief both sanction "accept and honor, since `PunchHole` zero-overwrites rather than leaving a hole," and reads return zeros either way. Flagging only because the literal spec semantics of NO_HOLE ("must not use a hole") are arguably not met; the sanctioned reading is in the spec, so this is a plan-level interpretation to confirm, not a defect I can call.
- **No other user-biting changes found.** No `EOVERFLOW`/`ENOTSUP` is returned (Global Constraint, plan:22); no issue/PR numbers in comments (plan:24, grep clean); the default dispatch branch comment now names only CACHE/BLOCK_STATUS, matching the implemented set.

## Assessment

The functional deliverable is correct and complete against the brief: all 13 enumerated requirements and 9 of the 10 priority checks are satisfied, the 15 named tests are present and reported green, the check order and per-command error codes match the spec, and the two new commands route through the metadata punch so the reap cannot be silently skipped. The one substantive gap is a **plan-mandated test that the brief omitted** (`TestConnection_PayloadReadIsBoundedByRead`), which leaves the payload-`Read` half of the split unpinned — a ~15-line addition, not a redesign. The remaining findings are test-surface and parity notes.

**Task quality:** Needs fixes — add the plan-mandated `TestConnection_PayloadReadIsBoundedByRead` (or have the controller adjudicate the brief-vs-plan discrepancy); everything else is approved.
