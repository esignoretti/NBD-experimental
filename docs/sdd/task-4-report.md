# Task 4 report: connection loop and READ

**Status:** DONE_WITH_CONCERNS
**Commit:** `042ead73a0dc32a798fe39c5451e83b746b74ffa` — `feat(nbd): connection loop and READ handler` (SSH-signed, 1 `BEGIN SSH SIGNATURE` block, tree clean, no AI attribution — `test/no-ai-attribution.sh HEAD~1` reports "no AI attribution in 1 commit(s)").
**Base:** `8f0f64c3`

## What I built

### Prerequisite widening (Step 1's "do that widening first")

- `internal/adapter/common/normalize.go` — `normalizeBlockStoreError` exported as `NormalizeBlockStoreError`, doc comment extended to say a protocol adapter that maps a raw store error itself must call it first.
- `internal/adapter/common/write_payload.go` — added `type Committer interface { Flush(ctx, payloadID string) (*block.FlushResult, error); LocalDurable() bool; RemoteDurable() bool; RequireDurableCommit() bool }`; widened `CommitBlockStore`'s second parameter from `*engine.Store` to `Committer`. Added the `pkg/block` import.
- Rename call sites updated: `read_payload.go:60`, `write_payload.go:68`, `:122`, `:171`, and the prose mention in `doc.go:29`.
- `internal/adapter/nfs/v4/handlers/content_errmap_wiring_test.go:17` — one **comment** updated to the new name so the symbol stays greppable. The brief said this file "contains ONE COMMENT mentioning the name and zero calls" (Trap 6) — confirmed. Not listed in the brief's file list; a one-word comment rename, no behaviour.

Verification the brief asked for before writing new code:
```
go build ./...                                            -> clean
go test ./internal/adapter/common/                        -> ok 5.713s
go test ./internal/adapter/nfs/... ./internal/adapter/smb/... -> all ok
```
A type-only widening that broke a call site would have meant a call site was not passing `*engine.Store`. None was.

### `internal/adapter/nbd/connection.go` (new, 334 lines)

`Timeouts`, `BlockStore` (7 methods), `StoreResolver`, `Handler`, `NewHandler`, `TakeDegraded`, `authContext`, `Conn`, `NewConn`, `Conn.Serve`, `Handler.serve`, `writeReply`, `mapError` — all exactly as the brief specifies.

- `Conn.Serve` recovers panics and logs at `Error` with `debug.Stack()`; imports `internal/logger` (package, called as `logger.Error`) and `runtime/debug`.
- `Handler.serve`: handshake → resolve store once → loop. `SetReadDeadline` armed before each read, **skipped while draining** (so the base adapter's 100 ms interruption is not overwritten); a request that arrives while draining is answered `ErrESHUTDOWN` and the connection stays open.
- `mapError` calls `common.NormalizeBlockStoreError` **before** classifying; store cases tested before the context pair; `ErrEIO` for `engine.ErrStoreClosed` and `common.ErrNotDurableYet`, `ErrENOSPC`/`ErrEPERM` from the `merrs` codes, `ErrESHUTDOWN` for the context pair, `ErrEIO` default. Never returns `ErrEOVERFLOW`/`ErrENOTSUP`.
- `internal/adapter/nbd/reply.go` — removed the `//nolint:unused` directive on `writeSimpleReply`, which is now the first caller.

### `internal/adapter/nbd/handlers_read.go` (new, 88 lines)

Checks in the required order: overflow → zero length → past end → flags. Overflow is a **subtraction** (`uint64(req.Length) > math.MaxUint64-req.Offset`), not a wrapping add, and returns `ErrEIO`. `buf` is allocated only after the bounds checks. `common.WithRequestDeadline` wraps the context per command.

### Adapter wiring (Step 6 + 7)

- `pkg/adapter/nbd/adapter.go` — `handler atomic.Pointer[nbd.Handler]` (needs `sync/atomic`); `SetRuntime` keeps `a.BaseAdapter.SetRuntime(rt)`; `Serve` builds the exporter, the resolver over `common.ResolveForWrite`, and the draining closure, then stores the handler and calls `ServeWithFactory`; `NewConnection` returns `nbd.NewConn(a.handler.Load(), conn)`. The Task 3 placeholder `nbdGreeting`/`nbdConn` and the now-unused `io` import are gone.
- `pkg/adapter/nbd/config.go` — `Config.timeouts()` narrowing four durations to three.
- `pkg/adapter/nbd/healthcheck.go` — the `Degraded` branch, before the delegation, using `TakeDegraded`.

### Tests

- `internal/adapter/nbd/connection_test.go` (new, 771 lines, `package nbd`) — 20 test functions. Uses the existing `pipe(t)` from `handshake_test.go` (not redefined, no `net.Pipe`), `fakeExporter`, `readGreeting`, `sendClientFlags`, `sendOption`, `goPayload`. Adds `storeFake` (with `panicOnRead` and the three durability accessors), the shared handshake/reply helpers, and every test the brief names.
- `internal/adapter/nbd/integration_test.go` (new, 214 lines) — `newIntegrationFixture` with `Enabled: true`, and the three integration tests.
- `pkg/adapter/nbd/healthcheck_test.go` — added `TestNBDAdapter_Healthcheck_NotDegradedWithoutFailures`.

## Exact commands and results

| Command | Result |
|---|---|
| `go build ./...` | clean |
| `go vet ./internal/adapter/nbd/ ./pkg/adapter/nbd/ ./internal/adapter/common/` | clean |
| `gofmt -l` on the touched trees | empty |
| `go test ./internal/adapter/nbd/ ./pkg/adapter/nbd/ ./internal/adapter/common/ -race -count=1 -timeout 240s` | all `ok` (4.774s / 4.550s / 8.409s) |
| `go test ./internal/adapter/nfs/... ./internal/adapter/smb/... -timeout 300s` | all `ok` (widening did not break a call site) |
| `GOTOOLCHAIN=go1.26.0 $HOME/go/bin/golangci-lint run ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/...` | `0 issues` |
| `bash test/no-ai-attribution.sh HEAD~1` | `no AI attribution in 1 commit(s)` |

`go test ./internal/adapter/nbd/ -race -v` passes 44 tests (20 new dispatch/unit + 3 integration + the 21 pre-existing handshake/protocol/volume tests).

## TDD RED evidence (Step 2)

Before implementing `connection.go`/`handlers_read.go`:
```
$ go test ./internal/adapter/nbd/ -run TestConnection -v -timeout 120s
# github.com/marmos91/dittofs/internal/adapter/nbd [...]
internal/adapter/nbd/connection_test.go:136:21: undefined: BlockStore
internal/adapter/nbd/connection_test.go:136:33: undefined: StoreResolver
internal/adapter/nbd/connection_test.go:142:21: undefined: Timeouts
internal/adapter/nbd/connection_test.go:148:69: undefined: Handler
internal/adapter/nbd/connection_test.go:149:9: undefined: NewHandler
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd [build failed]
```
Matches the brief's expected `undefined: NewHandler`, `undefined: BlockStore`.

## Step 10: the two bite proofs

**Proof 1 — overflow check.** Commented out the overflow guard (`if false && uint64(req.Length) > math.MaxUint64-req.Offset`):
```
=== RUN   TestConnection_OffsetPlusLengthOverflowIsEIO
    connection_test.go:537: overflowing READ reply error = 0, want ErrEIO (5)
--- FAIL: TestConnection_OffsetPlusLengthOverflowIsEIO (0.00s)
FAIL
```
Fails **on the assertion** at `connection_test.go:537`, not a compile error. Restored.

**Proof 2 — exact-end check.** Changed `>` to `>=` in the range check:
```
=== RUN   TestConnection_ReadAtExactEndIsServed
    connection_test.go:481: READ ending exactly at the export end returned error 22, want 0: the last byte is addressable
--- FAIL: TestConnection_ReadAtExactEndIsServed (0.00s)
FAIL
```
Fails **on the assertion** at `connection_test.go:481`. Restored.

**Bonus proof — Trap 7 (normalization is load-bearing).** Replaced `normalized := common.NormalizeBlockStoreError(err)` with `normalized := err`:
```
    connection_test.go:692: mapError(journal: local store full, all segments pinned by unsynced bytes) = 5, want 28
    connection_test.go:692: mapError(write: journal: local store full, all segments pinned by unsynced bytes) = 5, want 28
--- FAIL: TestMapError_ClassifiesStoreAndContextErrors
```
A disk-full condition reports `ErrEIO` (5) instead of `ErrENOSPC` (28) without normalization — exactly the bug class the design exists to prevent. Restored.

## Panic-recover failure mode (Step 10, third item)

Removed the `defer func() { recover() ... }()` block from `Conn.Serve` (and the now-unused `runtime/debug` import) and ran `TestConn_PanicInHandlerDoesNotKillTheProcess`:
```
=== RUN   TestConn_PanicInHandlerDoesNotKillTheProcess
panic: storeFake: ReadAt panicked on purpose

goroutine 6 [running]:
github.com/marmos91/dittofs/internal/adapter/nbd.(*storeFake).ReadAt(...)
	internal/adapter/nbd/connection_test.go:60 +0x120
github.com/marmos91/dittofs/internal/adapter/nbd.(*Handler).handleRead(...)
	internal/adapter/nbd/handlers_read.go:81 +0x174
github.com/marmos91/dittofs/internal/adapter/nbd.(*Handler).serve(...)
	internal/adapter/nbd/connection.go:221 +0x2b4
github.com/marmos91/dittofs/internal/adapter/nbd.(*Conn).Serve(...)
	internal/adapter/nbd/connection.go:157 +0x5c
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	0.639s
```
**Observed: the test binary CRASHES** (an uncaught `panic:` unwinding the goroutine, whole package `FAIL`), not a failed assertion. That is the correct failure mode for this test, as the brief states. Restored.

## Deviations from the brief

Two, both additive, both because the brief's own text and its code block disagree with the sibling adapters.

### 1. `Conn.Serve` also closes the socket (brief's Step-3 code block omits it)

The brief's verbatim `Conn.Serve` block has only the recover defer and the `serve` call. Neither the base adapter's accept loop (`pkg/adapter/base.go:373-397` unregisters the connection but never calls `Close`; its `forceCloseConnections` runs only when the drain deadline expires) nor `handler.serve` closes it. So the server's end of every finished connection would sit in `CLOSE_WAIT` until the finalizer ran. Both siblings close in their deferred connection-close handler: NFS at `pkg/adapter/nfs/connection.go:460`, SMB at `pkg/adapter/smb/connection.go:501`. Task 3's placeholder `nbdConn.Serve` also had `defer func() { _ = c.conn.Close() }()` — so dropping it would have been a regression against the code this task replaces. I added `defer func() { _ = c.conn.Close() }()` **after** the recover defer, so a panic is logged before the socket goes away.

The brief mentions closing in no other place (grep for `Close()` in `task-4-brief.md` returns nothing), so this is an omission rather than a conflict. It does not affect any test — `pipe(t)` closes both ends in `t.Cleanup`, and a client that sees EOF or a close is indistinguishable for the assertions in play.

### 2. `DISC` is handled before the draining check (brief's Step-3 ordering would reply to a DISC)

The brief's Step 3 puts the draining check (item 3, "when draining, answer new commands with `ErrESHUTDOWN`") ahead of the `CmdDisc` rule (item 5, "return nil without writing a reply"), and item 5 states the spec rule absolutely: *"There is no reply to an NBD_CMD_DISC."* With the brief's ordering, a DISC that arrives in the drain window gets an `ESHUTDOWN` reply header — a reply to a command that has no reply, in any state. I hoisted the DISC branch above the draining check, so a zero-offset/length DISC always returns `nil` with nothing written and a malformed one is still `ErrEINVAL`. This is the only ordering change; every other branch keeps the brief's order.

Neither deviation is a new product or architecture decision: both restore behaviour the brief's prose already requires (a reply-free DISC; a connection that is actually released) and that both sibling adapters already implement.

### 3. Added one test the brief did not list: `TestConnection_DrainingAnswersESHUTDOWN`

The brief specifies the drain behavior in detail (Step 3, five bullet points) and names no test for it, leaving the most subtle part of the loop uncovered. `draining` is injectable precisely so this is reachable. The test drives a READ while draining and asserts `ErrESHUTDOWN`, the cookie echo, that the store saw **zero** reads (answered, not served), and that a second command is still answerable (the connection was not closed on the first observation). It pins the two behaviours the brief is most emphatic about.

### 4. `mapError` logging

The brief says "put the original error in a log line at Debug for expected classes and Error otherwise". I log `Debug` for store-closed, not-durable-yet, no-space, permission-denied and the context pair, and `Error` for the default (unmapped engine fault). The brief's own classification table is what "expected" means here.

### 5. `a.Shutdown` rather than `a.BaseAdapter.Shutdown`

The brief's `draining` closure selects on `a.BaseAdapter.Shutdown`. Package-scoped `golangci-lint` flags that as `QF1008: could remove embedded field "BaseAdapter" from selector`. `Shutdown` is a promoted field with no override on `Adapter`, so `a.Shutdown` is the same field; I used it and noted why in a comment. The wire layer's `Timeouts`/`Handler`/`Conn` shape is unchanged.

## Concerns

1. **`Timeouts.Idle` is stored, plumbed and never used.** The brief declares the field (`Timeouts{Read, Write, Idle}`), defines it ("`Idle` bounds a connection that has completed its handshake but sends nothing, so `Read` must be the *shorter* of the two or a stalled client is held open by the wrong timer"), has `Config.timeouts()` carry it, and then specifies exactly one deadline in the loop — `SetReadDeadline(time.Now().Add(h.timeouts.Read))` before each read — with no `Idle` anywhere in Step 3, Step 6, or any test. So the field is dead in this task: the read deadline does bound an idle connection, but by `Read` (5 m default) rather than by `Idle` (also 5 m default). The brief's own sentence says `Read` must be *shorter* than `Idle` for the two to mean different things, and the defaults make them equal, so the distinction currently cannot be observed. I implemented exactly what Step 3 specifies rather than inventing an `Idle` deadline the brief does not ask for, and I am flagging it rather than guessing: either `Idle` should arm the pre-read deadline (which changes the partial-header timeout behaviour `TestConnection_PartialHeaderTimesOut` pins) or the field should not exist yet. **This needs a controller ruling before Task 5 builds on the same struct.** The SMB precedent the brief cites for `TimeoutsConfig` (`pkg/adapter/smb/connection.go:230`, `:456`) does use `Idle` for exactly this.

2. **`storeFake` is not concurrency-safe**, and neither is the handler. That matches the design (`Handler` is shared but stateless, and the dispatch is one command at a time per connection), and `-race` is clean across the suite. Noting it so Task 5 does not assume the fake can be driven from two goroutines.

3. **`TestConnection_FailedStoreResolutionDegradesAndCounts` does not assert the resolver error is surfaced**, only that `serve` returns non-nil and the counter moved. That is what the brief asks for and the ordering it insists on (error → count → consumed) is enforced. The error text itself is not pinned, so a future change to the wrapping message would not fail the test.

4. **Deferred minors untouched**, as instructed: the `createNBDAdapter` mapstructure gap, `UID == 0`, `UpdateAdapter`'s listener reuse, and the `""` subtest label. The first is now more consequential than when it was ledgered: `Serve` reads `a.config.timeouts()` and `a.config.UID`, both of which are always the compiled-in defaults because `createNBDAdapter` never populates them from the DB row. I did not fix it — the brief's Step 6 specifies exactly the four keys it reads, and the same gap exists for SMB today.

5. **The `handler` field is `atomic.Pointer[nbd.Handler]` and `-race` is clean**, but no test drives `Healthcheck` concurrently with `Serve` — the race the brief's Trap 1 describes was verified by the controller's probe before dispatch, not re-verified here. `TestNBDAdapter_Healthcheck_NotDegradedWithoutFailures` starts `Serve` and probes afterwards, which is the observable contract the brief asks for.
