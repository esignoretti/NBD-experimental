# Task 5 report — WRITE, FUA, and FLUSH

Status: DONE_WITH_CONCERNS
Commit: `6fa295e3` `feat(nbd): WRITE, FUA, and FLUSH over the existing durability seam`
Branch: `nbd-block-protocol` (not switched, not pushed)
Diff base for review: `7b284dfc` (see "Review base" below)
Diff: 5 files, +804 / -27

## What was implemented

| File | Change |
|---|---|
| `internal/adapter/nbd/handlers_write.go` | new — `handleWrite`, `rejectWrite` |
| `internal/adapter/nbd/handlers_control.go` | new — `handleFlush`, `commit` |
| `internal/adapter/nbd/connection.go` | `CmdWrite`/`CmdFlush` dispatch arms; `mapError` gains the `ErrAccessDenied` arm |
| `internal/adapter/nbd/connection_test.go` | 11 WRITE/FLUSH tests, `writeThroughClient`/`flushThroughClient`/`serveHandler` helpers, `storeFake.onFlush`, delete `TestConnection_WriteIsEINVALBeforeTask5`, fix `TestConnection_WriteWithNoHoleFlagIsEINVAL` |
| `internal/adapter/nbd/integration_test.go` | 4 integration tests, `newIntegrationFixtureAs` split, `writeVolumeFixture` |

Check order in `handleWrite` is flags → read-only → zero length → overflow → past end
(ENOSPC) → read payload → `PrepareWrite` → `WriteAt` → `CommitWrite` → FUA commit →
reply. `rejectWrite` replies first, then re-arms the read deadline from
`Timeouts.Read` and drains `req.Length` bytes with `io.CopyN(io.Discard, ...)`.

## TDD evidence

The implementation was written before the tests (mid-run steering), so RED was
reconstructed by reverting the production change at the source level and running
the tests against it. Two independent RED runs were captured, because there are
two independently observable production changes.

### RED run 1 — dispatch reverted (WRITE/FLUSH fall into `default` → EINVAL)

Reverting only the `case CmdWrite:` / `case CmdFlush:` arms from the switch in
`connection.go` and running
`go test ./internal/adapter/nbd/ -run 'TestConnection_Write|TestConnection_Flush' -v`:

```
--- PASS: TestConnection_WriteWithNoHoleFlagIsEINVAL (0.00s)
    connection_test.go:819: WRITE reply error = 22, want 0
--- FAIL: TestConnection_WritePersistsBytes (0.09s)
    connection_test.go:849: WRITE past the export end reply error = 22, want ErrENOSPC (28)
--- FAIL: TestConnection_WritePastEndIsENOSPC (0.00s)
--- PASS: TestConnection_WriteZeroLengthIsEINVAL (0.10s)
--- PASS: TestConnection_WriteTruncatedPayloadIsAnError (0.00s)
    connection_test.go:1023: the FUA write never reached Flush
--- FAIL: TestConnection_WriteWithFUAIsDurableBeforeReply (2.08s)
    connection_test.go:1062: FLUSH on a durable store reply error = 22, want 0
--- FAIL: TestConnection_FlushIsAckedWhenDurable (0.00s)
    connection_test.go:1082: FLUSH on a non-durable store reply error = 22, want ErrEIO (5)
--- FAIL: TestConnection_FlushIsEIOWhenNotDurable (0.00s)
    connection_test.go:1103: FLUSH after a malformed one reply error = 22, want 0
--- FAIL: TestConnection_FlushWithNonZeroLengthIsEINVAL (0.00s)
    connection_test.go:1120: FLUSH on a read-only export reply error = 22, want 0
--- FAIL: TestConnection_FlushOnReadOnlyExportIsPermitted (0.00s)
FAIL
```

The same revert also fails all four new integration tests on assertions
(`integration_test.go:301, :333, :370, :400`). Every failure is an assertion
failure, never a compile error.

Three tests pass under RED by design and that is expected, not a gap:
`WriteWithNoHoleFlagIsEINVAL`, `WriteZeroLengthIsEINVAL` and
`WriteTruncatedPayloadIsAnError` assert an `EINVAL` reply / a returned error, and
the `default` branch answers `EINVAL` too. They are not the RED carriers for this
task; they are the regression pins for check ordering.

### RED run 2 — `mapError` arm reverted only

With the dispatch restored and only `case merrs.ErrPermissionDenied, merrs.ErrAccessDenied:`
reverted to `case merrs.ErrPermissionDenied:`,
`go test ./internal/adapter/nbd/ -run TestIntegration_WriteDeniedByPermissionIsEPERM -v`:

```
    integration_test.go:301: WRITE refused by file permissions reply error = 5, want ErrEPERM (1)
--- FAIL: TestIntegration_WriteDeniedByPermissionIsEPERM (0.08s)
FAIL
```

This is the supervisor's finding reproduced exactly: a write refused by POSIX
mode bits reached the client as `NBD_EIO` (5), a hardware-failure report for an
authorization decision.

### GREEN

`go test ./internal/adapter/nbd/ -race -v` after restoring both: **58 `--- PASS`,
0 `--- FAIL`**, `ok github.com/marmos91/dittofs/internal/adapter/nbd 7.050s`.
(58 includes the pre-existing Task 1–4 tests; the 15 added by this task are
listed below.)

## Step 8 — FUA bite proof

The FUA commit was moved to run *after* the reply (as a `defer` after
`writeSimpleReply`), then:

```
$ go test ./internal/adapter/nbd/ -run TestConnection_WriteWithFUAIsDurableBeforeReply -v
...
    connection_test.go:1038: the reply was written before the FUA commit: a client that has seen the reply has not been given the durability it asked for
--- FAIL: TestConnection_WriteWithFUAIsDurableBeforeReply (0.10s)
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	1.200s
```

It fails **on the assertion at `connection_test.go:1038`**, not on a compile
error. The call was then restored and the test re-run green. The proof is real:
the test observes the server blocked inside `storeFake.Flush` and asserts that no
reply header arrives on the client within 250 ms.

## Verification

| Command | Result |
|---|---|
| `go test ./internal/adapter/nbd/ -race -v` | pass — 58 PASS / 0 FAIL, `ok ... 7.050s` |
| `go build ./...` | pass |
| `go vet ./internal/adapter/nbd/` | pass |
| `gofmt -l internal/adapter/nbd/` | no output (clean) |
| `GOTOOLCHAIN=go1.26.0 $(go env GOPATH)/bin/golangci-lint run ./internal/adapter/nbd/...` | pass — `0 issues.` |
| `bash test/no-ai-attribution.sh 55a4386a` | pass — "no AI attribution in 5 commit(s)" |

Note on the linter: it is not on `PATH` in a bare shell (`which golangci-lint`
exits 1); the binary is at `$(go env GOPATH)/bin/golangci-lint`. `GOTOOLCHAIN=go1.26.0`
is required, as the brief said — the ambient toolchain here is go1.27.2.

The signed commit is `6fa295e3`. It carries a `gpgsig` header
(`git cat-file commit HEAD | grep -c '^gpgsig'` → 1); `git log --show-signature`
reports "No signature" only because `gpg.ssh.allowedSignersFile` is unset in this
repo, so git cannot *verify* it. Signing used the documented
`git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S` form.

## Tests added (15)

Connection (`connection_test.go`), all 11 named in the brief:
`TestConnection_WritePersistsBytes`, `_WritePastEndIsENOSPC`,
`_WriteZeroLengthIsEINVAL`, `_RejectedWriteKeepsTheStreamSynchronized`,
`_RejectedHugeWriteDoesNotAllocate`, `_WriteTruncatedPayloadIsAnError`,
`_WriteWithFUAIsDurableBeforeReply`, `_FlushIsAckedWhenDurable`,
`_FlushIsEIOWhenNotDurable`, `_FlushWithNonZeroLengthIsEINVAL`,
`_FlushOnReadOnlyExportIsPermitted`.

Integration (`integration_test.go`): `TestIntegration_FlushWithCommitAckJournalSucceeds`,
`TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO`,
`TestIntegration_WriteThenFlushThenReadBackByteExact`, plus
`TestIntegration_WriteDeniedByPermissionIsEPERM` for the `ErrAccessDenied` arm.

Deleted: `TestConnection_WriteIsEINVALBeforeTask5` (pinned the `default` branch for
a command now served). Updated, not deleted:
`TestConnection_WriteWithNoHoleFlagIsEINVAL` now sends its declared 4096-byte
payload — without it the server's drain blocks and the test hangs rather than
fails.

## Concerns

1. **The brief's `TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO`
   recipe cannot produce the failure it names.** The brief says
   `bs.SetRequireDurableCommit(true)` plus a memory remote should yield `ErrEIO`.
   It cannot, on this fixture: `testruntime.NewWithShareStore` opens a
   journal-backed local tier, and `engine.Store.Flush`
   (`pkg/block/engine/flush.go:554`) returns on the `LocalDurable()` fast path
   *before* the remote is consulted. Strict mode alone is a no-op here. The test
   as committed additionally forces the local store non-durable via
   `bs.Local().(interface{ SetDurable(bool) }).SetDurable(false)` — the same lever
   `internal/adapter/common/commit_durability_test.go` uses — which makes the
   commit land on `Finalized=true && remoteDurable=false` → `ErrNotDurableYet` →
   `ErrEIO`. Both preconditions (`!LocalDurable()`, `!RemoteDurable()`) are
   asserted in the test rather than assumed. Flagged, not silently patched: the
   intent (an unachievable durability promise must not be acknowledged) is
   preserved, the brief's mechanism is not.

2. **The `ErrAccessDenied` → `ErrEPERM` arm was a real bug in Task 4's code**, found
   by the supervisor mid-run, not by me. `mapError` handled only
   `ErrPermissionDenied`; the metadata permission funnel returns `ErrAccessDenied`
   for every denial by POSIX mode bits, so every permission refusal reached the
   client as `NBD_EIO`. Fixed here with a comment naming both POSIX codes and why
   NBD collapses them, and covered by
   `TestIntegration_WriteDeniedByPermissionIsEPERM`.

3. **`TestConnection_RejectedHugeWriteDoesNotAllocate` is weaker than it reads.**
   It asserts `TotalAlloc` grew by less than the declared 1 GiB. Go's allocator
   would not have grown by 1 GiB for a `make([]byte, 1<<30)` that is immediately
   discarded either — the assertion catches the gross case, not a modest one. The
   real guard against an unbounded buffer is structural (`io.CopyN` to
   `io.Discard`, no `make` on the rejection path) and is visible in the diff. The
   test still does real work: it pins that the reply arrives within the request
   budget while the client is stalled, which is exactly the property the
   reply-before-drain ruling exists to provide.

4. **My first fixture fix was wrong and the supervisor corrected it.** I initially
   chmod'd the volume to 0o666 to get past `PrepareWrite`; that made the tests
   pass while hiding the identity mismatch the supervisor identified
   (`ApplyOwnerDefaults` sets the file's owner to the *creating* identity, so a
   fixture that creates as 1000 and writes as 65534 is refused — correctly). The
   committed fixture matches the creating identity to `integrationIdentity` and
   keeps the mismatched case as its own test. No `SetFileAttributes` mode hack
   remains.

5. **`newWriteTestHandler(t, st, exp) *Handler` from the brief does not exist as
   named.** The brief specifies that signature; the fixture shape that works is a
   `writeVolumeFixture` carrying the resolved export and the runtime, with
   `(f *writeVolumeFixture) newHandler(t, st)` — because the handler needs the
   *resolved* export (real file handle, real payload ID) and the runtime's
   metadata service together, and passing an arbitrary `*Export` separately would
   let a test pair a fake export with a real service and get an opaque
   `ErrUnknownExport` at `PrepareWrite` time. The intent — a real metadata
   service, the fixture factored so the two files cannot drift — is met; the
   parameter list differs.

## Review base

The review diff base is `7b284dfc`, not the `55a4386a` named at dispatch. Two
plan-only commits landed while I worked — `55a4386a` ("reply before draining a
rejected write's payload") and `7b284dfc` ("map ErrAccessDenied to EPERM, and pin
the fixture's identity") — and my commit sits on top of them. Both are
`docs/superpowers/plans/...` only; the review diff
`git diff 7b284dfc HEAD` is 5 files, all under `internal/adapter/nbd/`, +804/-27,
with no plan changes in it. `55a4386a` is an ancestor of `HEAD`.
