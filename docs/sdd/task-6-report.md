# Task 6 report — TRIM, WRITE_ZEROES, and read-only exports

**Status:** DONE_WITH_CONCERNS (one required change to a pre-existing committed test — see Concerns 1)

**Commits:**

- `7473656d` — `feat(nbd): TRIM and WRITE_ZEROES via PunchHole` — the task itself.
- `ac29d710` — `test(nbd): bound the payload read and the punch failure arm` — round-1 review fixes, below.

Both signed, conventional, no attribution (`test/no-ai-attribution.sh a533e7f5` → `no AI attribution in 5 commit(s)`). Each carries an ssh `gpgsig` header; `%G?` reports `N` because `gpg.ssh.allowedSignersFile` is unset on this machine, which is true of every commit on this branch (environmental, not a missing signature).

**Diff:** `7473656d` — 5 files, +687/-16 (`internal/adapter/nbd/handlers_write.go`, `internal/adapter/nbd/connection.go`, `internal/adapter/nbd/connection_test.go`, `internal/adapter/nbd/integration_test.go`, `pkg/adapter/nbd/config.go`). `ac29d710` — 1 file, +56/-1 (`internal/adapter/nbd/connection_test.go`).

## Round 1 — review fixes (`ac29d710`)

Four items were ruled in scope; three further findings were explicitly ruled out of scope and are deliberately untouched.

**(1) `TestConnection_PayloadReadIsBoundedByRead` added.** A WRITE header declaring 4096 bytes, 100 bytes of payload, then silence on an **open** connection, with `Timeouts{Read: 200ms, Idle: 5s}` — the inverse of the header test, so a payload read that inherited the header's Idle budget is still waiting at the 1s mark. Asserts `serve` returns non-nil and `st.writes == 0`. Written against `newTestHandler` alone (no metadata service needed: the deadline fires long before `PrepareWrite`), so no `newWriteTestHandler` re-export was required.

RED/GREEN for it, both halves:

```
$ # arm at internal/adapter/nbd/handlers_write.go:70 removed
$ go test ./internal/adapter/nbd/ -count=1
--- FAIL: TestConnection_PayloadReadIsBoundedByRead (1.00s)
    connection_test.go:1042: serve did not return within 1s of a 200ms read timeout: the payload read is not bounded by Timeouts.Read
FAIL
github.com/marmos91/dittofs/internal/adapter/nbd	5.959s
```

It is the **only** failure: the same run lists no other failing test, so with the arm gone the rest of the package — including the committed `TestConnection_WriteTruncatedPayloadIsAnError`, which uses equal budgets and closes the client, so EOF ends its read before any deadline is consulted — is green. Restoring the arm returns the package to `ok ... 4.903s`.

**(2) Stale failure message fixed** at `connection_test.go:738`: `"serve did not return within 2s of a 200ms read timeout"` → `"serve did not return within 2s of a 200ms idle timeout: Timeouts.Idle is not armed"`. The assertion was already right; only the string named the wrong timer.

**(3) `TestConnection_TrimPunchFailureIsEIO` added.** Sets `st.punchErr = errors.New("boom")` against a real metadata fixture, sends a TRIM, asserts the reply is `NBD_EIO` and that the store was reached once (`st.punches == 1`). This exercises the `bs.PunchHole` failure branch of `handleDiscard` (`handlers_write.go:248-250`) that `punchErr` was declared for and nothing set. The field stays, per the ruling.

**(4) This section.** 

Ruled out of scope and **not** touched: (A) no FUA-on-TRIM / FUA-on-WRITE_ZEROES test (deferred minor — the plan's Verification Checklist has no such row); (B) no `res.PayloadID != ""` guard at `handlers_write.go:248` (deliberate omission, recorded: NBD resolves `Export.PayloadID` once at handshake from a regular file and `buildPayloadID` never returns empty, so the guard is unreachable on NBD's own paths; no comment added claiming so); (C) no `common.WithRequestDeadline` around `bs.PunchHole` (the sibling framing was wrong — `internal/adapter/nfs/v4/handlers/deallocate.go:93` passes the bare `ctx.Context` too, and `handleDiscard` matches the only sibling that punches).

## What landed

- `handleTrim` and `handleWriteZeroes`, both delegating to one shared `handleDiscard(ctx, bs, exp, req, conn, allowed CommandFlags, outOfRange NBDError)`. The two commands differ only in the flag mask and the out-of-range code (EINVAL for TRIM, ENOSPC for WRITE_ZEROES), so those two values are parameters and everything else is one implementation.
- Check order exactly as the brief specifies: flags → read-only → zero length → overflow → `offset >= size` → `offset+length > size` → metadata punch → block-store punch → FUA commit → reply. No clamp anywhere.
- `storeFake` gained `punches int` and `punchErr error`; `PunchHole` increments `punches` first and returns `punchErr` before zero-filling.
- Dispatch: `CmdTrim` → `handleTrim`, `CmdWriteZeroes` → `handleWriteZeroes`. `CmdCache`/`CmdBlockStatus` still fall to `default` → `ErrEINVAL`; the default's comment was updated because it claimed TRIM/WRITE_ZEROES were unimplemented.
- `Timeouts.Idle` arm in the dispatch loop, inside `if !h.draining()`: `budget := h.timeouts.Read; if h.timeouts.Idle > 0 { budget = h.timeouts.Idle }`.
- `Read` armed before the WRITE payload read in `handleWrite` (previously the payload inherited the header read's deadline).
- New test helper `readOnlyExport(size)` alongside `testExport(size)` — a second constructor rather than mutating a shared `Export`.

## Verification

Exact command and result (Step 9):

```
go build ./... && go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/...
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	10.589s
ok  	github.com/marmos91/dittofs/pkg/adapter/nbd	3.749s
ok  	github.com/marmos91/dittofs/internal/adapter/common	8.629s
```

`gofmt -l internal/adapter/nbd/` → empty. `go vet ./internal/adapter/nbd/` → clean.
`GOTOOLCHAIN=go1.26.0 golangci-lint run ./internal/adapter/nbd/... ./pkg/adapter/nbd/...` → `0 issues.`

Re-run after the round-1 fixes (`ac29d710`):

```
$ go build ./... && go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/...
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	9.560s
ok  	github.com/marmos91/dittofs/pkg/adapter/nbd	(cached)
ok  	github.com/marmos91/dittofs/internal/adapter/common	(cached)
```

`gofmt -l` empty, `go vet` clean, package lint `0 issues.`

All 15 tests the brief names exist and pass:

```
--- PASS: TestConnection_TrimIsEPERMOnReadOnlyExport (0.00s)
--- PASS: TestConnection_TrimThenReadIsZeros (0.08s)
--- PASS: TestConnection_TrimPastEndIsEINVAL (0.00s)
--- PASS: TestConnection_TrimOffsetPastEndIsEINVAL (0.00s)
--- PASS: TestConnection_WriteZeroesPastEndIsENOSPC (0.00s)
--- PASS: TestConnection_WriteZeroesWithNoHoleIsAccepted (0.06s)
--- PASS: TestConnection_TrimZeroLengthIsEINVAL (0.00s)
--- PASS: TestConnection_WriteZeroesZeroLengthIsEINVAL (0.00s)
--- PASS: TestConnection_WriteZeroesThenReadIsZeros (0.08s)
--- PASS: TestConnection_WriteZeroesOnReadOnlyExportIsEPERM (0.00s)
--- PASS: TestConnection_ReadOnReadOnlyExportSucceeds (0.00s)
--- PASS: TestConnection_WriteOnReadOnlyExportIsEPERM (0.00s)
--- PASS: TestConnection_IdleDeadlineClosesASilentConnection (0.20s)
--- PASS: TestIntegration_TrimFreesAndReadsZero (0.18s)
--- PASS: TestIntegration_WriteZeroesDoesNotMaterializeChunks (0.13s)
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	1.521s
```

The two round-1 additions, from the same package:

```
--- PASS: TestConnection_PayloadReadIsBoundedByRead (0.20s)
--- PASS: TestConnection_TrimPunchFailureIsEIO (0.08s)
```

Advertised commands now equal served commands: `TransmissionFlags()` sets SEND_FLUSH, SEND_FUA, SEND_TRIM, SEND_WRITE_ZEROES, and the dispatch serves READ, WRITE, FLUSH, TRIM, WRITE_ZEROES (plus the mandatory DISCONNECT). CACHE and BLOCK_STATUS are advertised by neither and answered by neither.

Step 2 (RED, implementation stashed, tests present):

```
--- FAIL: TestConnection_TrimIsEPERMOnReadOnlyExport (0.00s)
--- FAIL: TestConnection_TrimThenReadIsZeros (0.11s)
--- FAIL: TestConnection_WriteZeroesPastEndIsENOSPC (0.00s)
--- FAIL: TestConnection_WriteZeroesWithNoHoleIsAccepted (0.10s)
--- FAIL: TestConnection_WriteZeroesThenReadIsZeros (0.11s)
--- FAIL: TestConnection_WriteZeroesOnReadOnlyExportIsEPERM (0.00s)
--- FAIL: TestConnection_IdleDeadlineClosesASilentConnection (2.00s)
```

(The two zero-length tests, the two out-of-range TRIM tests and the two read-only READ/WRITE tests are green under the stash: they exercise the pre-existing EINVAL default branch and Task 5's checks, so they were never RED. They are still the required pins.)

## Step 8 — the bite proof, verbatim

`handleWriteZeroes` replaced with the zero-buffer `WriteAt`, then:

```
$ go test ./internal/adapter/nbd/ -run 'TestConnection_WriteZeroesThenReadIsZeros|TestIntegration_WriteZeroesDoesNotMaterializeChunks' -v
=== RUN   TestConnection_WriteZeroesThenReadIsZeros
    connection_test.go:1376: the store saw 0 punches, want 1
--- FAIL: TestConnection_WriteZeroesThenReadIsZeros (0.10s)
=== RUN   TestIntegration_WriteZeroesDoesNotMaterializeChunks
    integration_test.go:535: manifest has 1 blocks after WRITE_ZEROES, want fewer than the 1 before it: a zero-buffer WriteAt would materialize a new chunk
--- FAIL: TestIntegration_WriteZeroesDoesNotMaterializeChunks (0.10s)
FAIL
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	0.960s
FAIL
```

Both fail, as required. The `st.writes == 0` half of the unit assertion bites independently — with the two assertions in the opposite order it reports:

```
    connection_test.go:1376: the store saw 1 writes, want 0: WRITE_ZEROES must punch, not write a zero buffer
--- FAIL: TestConnection_WriteZeroesThenReadIsZeros (0.10s)
```

Implementation restored (`git diff` confirms `handleWriteZeroes` back to the punch pair) and both tests pass again.

## Concerns

1. **`TestConnection_PartialHeaderTimesOut` had to change its fixture, and that is a change to already-committed code.** As committed it used `Timeouts{Read: 200ms, Write: 5s, Idle: 5s}` and asserted that a client which sends four bytes of a 28-byte header and then stalls is dropped. Under the mandated split the header wait is bounded by `Idle`, not `Read`, so with `Idle = 5s` the connection is correctly held for 5 seconds and the test fails:
   `connection_test.go:733: serve did not return within 2s of a 200ms read timeout` — the exact failure the first `-race` run produced. The fixture is now `Timeouts{Read: 5s, Write: 5s, Idle: 200ms}` with a comment explaining that Idle is the timer that bounds a silent wait for a header. This is unavoidable if the split is real: a partial header *is* a silent connection. The brief anticipated the test's weakness ("the wrong way round to catch this") but not that fixing the split would invert it. Flagging for review rather than silently rewriting.
2. **Two doc comments were corrected** beyond the brief's list: the `Timeouts` type doc in `internal/adapter/nbd/connection.go` ("Read bounds reading one request header or payload … Read must be the shorter of the two") and the `TimeoutsConfig.Read` doc in `pkg/adapter/nbd/config.go`. Both described the pre-fix behaviour and would have been false after it — the same class of false doc the brief cites at `pkg/adapter/nbd/config.go:42`. Comment-only.
3. **The `Idle` arm has no `min()` against `Read`**, per the brief: `Idle == 0` means "no idle timeout", so folding it in would turn an unset Idle into an instant timeout. Consequence: an operator who sets `Idle > Read` gets an idle wait longer than the read budget, which is the NFS shape too (`pkg/adapter/nfs/connection.go:312`). The `Timeouts` doc no longer claims otherwise. The `decision:` marker the brief asks for on trap 9 is at the call site in `internal/adapter/nbd/connection.go`, recording both the reason and that consequence.
4. **`bs.PunchHole` is called with `ctx`, not a deadline-wrapped context.** RULED ON in round 1: **not fixed, and not to be fixed.** The reviewer's "diverges from every sibling store call" framing was checked and is wrong — `internal/adapter/nfs/v4/handlers/deallocate.go:93` passes the bare `ctx.Context` too, and `handleDiscard` matches the only sibling that punches. Left as it is.
5. **`res.PayloadID` is used unguarded.** RULED ON in round 1: **not fixed, and not to be fixed.** Deliberate omission, recorded: NBD's `Export.PayloadID` is resolved once at handshake from a regular file and `buildPayloadID` never returns empty, so the guard would be unreachable on NBD's own paths; the reachable path is a cross-protocol unlink race while the connection is open, low likelihood and no crash. No guard added and no comment added claiming unreachability. (NFS DEALLOCATE guards with `res.PayloadID != "" && length > 0 && offset < res.File.Size`.)
6. Integration fixtures are 1 MiB exports, and the punches cover the whole device. `metadata.Service.PunchHole` returns early on `len(file.Blocks) == 0`, so the assertion that the manifest was non-empty *before* the punch is load-bearing and is asserted (probe: 1 MiB → 1 block, 2 MiB → 2 blocks, 4 MiB → 4 blocks, `DrainAllUploads` after `Flush`).

## Deviations from the brief

- `TestConnection_PartialHeaderTimesOut` fixture change (Concern 1) — required by the split, reason given above.
- Two doc-comment corrections (Concern 2) — the brief's trap 9 names the false doc; these are the same defect in the two remaining places.
- `handleDiscard` is a shared helper rather than two independent handler bodies. The brief specifies two exported-shaped methods and both exist with the specified signatures; the shared body is an internal detail. All comments the brief asks for (`decision:` for traps 4, 5 and 9) are in place.

## Deferred to the whole-branch review (ruled out of scope in round 1)

- No FUA-on-TRIM / FUA-on-WRITE_ZEROES test. The branch `if req.Flags&FlagFUA != 0 { h.commit(...) }` in `handleDiscard` is therefore unexercised. The plan's Verification Checklist has no such row; the reviewer's finding goes to the ledger as a deferred minor.
- The reachable cross-protocol unlink race behind the unguarded `res.PayloadID` (Concern 5).
