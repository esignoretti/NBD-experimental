## Task 6: TRIM, WRITE_ZEROES, and read-only exports

**Files:**
- Modify: `internal/adapter/nbd/handlers_write.go`, `internal/adapter/nbd/connection.go`
- Test: `internal/adapter/nbd/connection_test.go`, `internal/adapter/nbd/integration_test.go`

**Interfaces:**
- Consumes: Task 4's `BlockStore`, Task 5's `handleWrite` structure.
- Produces:
  - `func (h *Handler) handleTrim(ctx context.Context, bs BlockStore, exp *Export, req *Request, conn net.Conn) error`
  - `func (h *Handler) handleWriteZeroes(ctx context.Context, bs BlockStore, exp *Export, req *Request, conn net.Conn) error`

- [ ] **Step 1: Write the failing tests**

```go
// storeFake needs a punches counter and a punchErr field before these tests can
// assert anything: the fake's PunchHole (connection_test.go:99) zero-fills and
// returns without recording that it ran, and Step 7's bite proof depends on
// that record existing. Add both in this task.
//
// Tests that expect Error == 0 on TRIM or WRITE_ZEROES need Task 5's
// writeVolumeFixture (integration_test.go:48) and its newHandler method
// (integration_test.go:69), which wire a real metadata service: the read-path
// helper passes nil for meta (connection_test.go:151), and the handler now
// calls h.meta.PunchHole. newWriteVolumeFixture(t).newHandler(t, st) is the
// call. Rejection tests may keep the nil-meta helper.

func TestConnection_TrimIsEPERMOnReadOnlyExport(t *testing.T) {
	// ErrEPERM. Note the export DOES advertise SEND_TRIM even when read-only:
	// handshake.go:101 sets it unconditionally and only adds READ_ONLY. So this
	// is not a client violating "MUST NOT send a trim request unless
	// NBD_FLAG_SEND_TRIM was set" — it is the case the spec's error list names
	// explicitly: "it SHOULD return NBD_EPERM if it receives a write or trim
	// request on a read-only export".
}

func TestConnection_TrimThenReadIsZeros(t *testing.T) {
	// Write 8192 non-zero bytes, TRIM [4096, 8192), read back: zeros in the
	// trimmed range, [0, 4096) untouched. Needs the real-meta fixture.
}

func TestConnection_TrimPastEndIsEINVAL(t *testing.T) {
	// Length runs past the export end → ErrEINVAL, refused not clamped. The
	// spec: "it SHOULD return NBD_EINVAL if it receives a read or trim request
	// including one or more sectors beyond the size of the device." READ
	// already answers EINVAL past the end (handlers_read.go:42), and the same
	// condition deserves the same answer.
}

func TestConnection_TrimOffsetPastEndIsEINVAL(t *testing.T) {
	// offset >= size with a small length. Asserted separately because it is the
	// case a subtraction-based clamp underflows on (see Step 3).
}

func TestConnection_WriteZeroesPastEndIsENOSPC(t *testing.T) {
	// ErrENOSPC, not EINVAL: the spec gives write zeroes its own rule — "The
	// server SHOULD return NBD_ENOSPC if it receives a write zeroes request
	// including one or more sectors beyond the size of the device."
}

func TestConnection_WriteZeroesWithNoHoleIsAccepted(t *testing.T) {
	// The spec MUSTs this: "The server MUST support the use of this flag if it
	// advertises NBD_FLAG_SEND_WRITE_ZEROES." Expect Error == 0 and
	// st.punches == 1 (PunchHole zero-overwrites rather than creating a hole).
}

func TestConnection_TrimZeroLengthIsEINVAL(t *testing.T) {
	// Length: 0 → ErrEINVAL, the same answer READ, WRITE and WRITE_ZEROES give.
}

func TestConnection_WriteZeroesZeroLengthIsEINVAL(t *testing.T) {
	// Length: 0 → ErrEINVAL, matching READ and WRITE. No payload either way, so
	// the connection must stay usable for the next command.
}

func TestConnection_WriteZeroesThenReadIsZeros(t *testing.T) {
	// The important one: WRITE_ZEROES must not materialize real chunks.
	// Assert st.punches == 1 and st.writes == 0 on the fake.
}

func TestConnection_WriteZeroesOnReadOnlyExportIsEPERM(t *testing.T) {
}

func TestConnection_ReadOnReadOnlyExportSucceeds(t *testing.T) {
	// The flag means writes are refused, not that the device is unusable.
}

func TestConnection_WriteOnReadOnlyExportIsEPERM(t *testing.T) {
	// Pins Task 5's handleWrite check 2. It lives here because the read-only
	// fixture is added in this task; the code under test is Task 5's.
}
```

These tests need a read-only export variant of the fixture: `testExport(size)`
(`connection_test.go:127`) returns `ReadOnly: false`. Add the flag as a parameter
or a second helper rather than mutating the returned `Export`, which other tests
share.

- [ ] **Step 2: Run them and confirm they fail**

Run: `go test ./internal/adapter/nbd/ -run 'TestConnection_Trim|TestConnection_WriteZeroes|TestConnection_ReadOnReadOnly|TestConnection_WriteOnReadOnly' -v`
Expected: FAIL.

- [ ] **Step 3: Implement both handlers**

Both call the same two-step sequence the NFS DEALLOCATE handler uses (`internal/adapter/nfs/v4/handlers/deallocate.go:67`, `:91`):

```go
authCtx := h.authContext(ctx, exp)
res, err := h.meta.PunchHole(authCtx, exp.Handle, req.Offset, uint64(req.Length))
if err != nil {
    return h.writeReply(conn, req.Cookie, mapError(err), nil)
}
if _, err := bs.PunchHole(ctx, string(res.PayloadID), res.PreOpBlocks, req.Offset, uint64(req.Length)); err != nil {
    return h.writeReply(conn, req.Cookie, mapError(err), nil)
}
```

**`mapError` returns `NBDError`, which is not an `error`** — it is a `uint32` with no `Error()` method (`internal/adapter/nbd/protocol.go:76`), so `return mapError(err)` from a handler does not compile and, if it did, would send the client nothing and hang it. Every error on this path goes through `h.writeReply(conn, req.Cookie, mapError(err), nil)`, which is what the shipped `handleWrite` and `handleRead` do (`handlers_write.go:88`, `handlers_read.go:84`). Use that shape; there is no helper that turns a store error into a returned error, and adding one would put the reply in two places.

**The metadata punch is not optional, for two independent reasons.**

First, **without it the storage is never freed.** `engine.Store.PunchHole` does the reap — `DecrementRefCountAndReapMany` plus `ReprojectBlocks` — only inside `if len(currentBlocks) > 0` (`pkg/block/engine/readwrite.go:308`). Called with `nil`, it skips the reap entirely and only zero-overwrites the range, so reads return zeros while every CAS object stays referenced and the manifest keeps listing the chunks. The command would look like it worked and free nothing, which is the one thing a discard exists to do. `metadata.Service.PunchHole` (`pkg/metadata/sparse.go:48`) prunes `file.Blocks`, persists the new manifest, and returns `PreOpBlocks` for the block store to reap against.

Second, **without it the command bypasses authorization.** `metadata.Service.PunchHole` calls `checkWritePermission` (`pkg/metadata/sparse.go:80`); `engine.Store.PunchHole` checks nothing. WRITE goes through `PrepareWrite`, which checks. A TRIM that went straight to the engine would be the one mutating command in the adapter that no permission funnel sees — and it destroys data. This is CLAUDE.md invariant 2: every operation carries a `*metadata.AuthContext`.

Order of checks, and the order is the point:

1. `req.Flags &^ allowedFlags != 0` → `ErrEINVAL` (see the flag table below).
2. `exp.ReadOnly` → `ErrEPERM`.
3. `req.Length == 0` → `ErrEINVAL`.
4. `req.Offset > math.MaxUint64 - uint64(req.Length)` → `ErrEIO`, the same wrap guard `handleRead` uses and for the same reason.
5. `req.Offset >= exp.Size` → `ErrEINVAL` for TRIM, `ErrENOSPC` for WRITE_ZEROES.
6. `req.Offset + uint64(req.Length) > exp.Size` → the same two codes.
7. The store calls above.
8. If `req.Flags&FlagFUA != 0`, `h.commit(ctx, bs, exp)` **before** writing the reply, exactly as WRITE does.
9. Write the reply with no payload.

**Out of range is refused, not clamped — and a clamp is where the bug was.** An earlier draft of this task clamped with `end := min(offset+length, size); length := end - offset`. That **underflows** when `req.Offset > exp.Size`: `end` becomes `exp.Size`, and `exp.Size - req.Offset` wraps to a number near 2⁶⁴. Measured: `offset=10000, length=100, size=8192` → `18446744073709549808`. The engine's own overflow guard then turns it into a confusing `punch range overflow` error instead of the right answer. Checks 5 and 6 make the subtraction unnecessary, so there is no clamp to get wrong.

This also resolves an internal contradiction in the design spec. §4.4 said `TRIM`/`WRITE_ZEROES` are *"clamped to the export end"* while §4.6 and §13 both said a `TRIM` past the end is `EINVAL`. The spec's per-command error guidance is the more specific rule and wins: *"it SHOULD return `NBD_EINVAL` if it receives a read or trim request including one or more sectors beyond the size of the device"*, and for write zeroes *"The server SHOULD return `NBD_ENOSPC` if it receives a write zeroes request including one or more sectors beyond the size of the device."* Both spec sections are corrected in the same commit as this plan edit.

**TRIM over-delivers, deliberately.** The spec says a client *"MUST NOT make any assumptions about the contents of the export affected by this command"* and a server *"MAY discard length bytes starting at offset, but is not required to."* `PunchHole` zeroes the range anyway (`pkg/block/engine/readwrite.go:281`). That is the honest mapping: the alternative is a cheaper discard leaving reads non-deterministic. Put this in a `decision:` comment at the call site with the spec quote, because the next reader will otherwise "optimize" it.

**WRITE_ZEROES must not become a zero-buffer `WriteAt`.** That is exactly the reference library's bug: on a CAS store it *materializes* zero chunks, so a filesystem's discard path allocates storage instead of freeing it. The test asserts `st.punches == 1` and `st.writes == 0`, and that assertion is the guard.

Flag validation differs by command: `TRIM` accepts `FUA` only; `WRITE_ZEROES` accepts `FUA` and `NO_HOLE`. Any other bit → `ErrEINVAL`, for the same reason as `WRITE` — the flag is not documented as applicable to that request. FUA on either is honored through the same `h.commit` path as WRITE.

**`NO_HOLE` on `WRITE_ZEROES` must be accepted, not rejected.** The spec: *"The server MUST support the use of this flag if it advertises `NBD_FLAG_SEND_WRITE_ZEROES`"*, and we do. It is the client asking that the range be genuinely allocated rather than left sparse. `PunchHole` already zero-overwrites the range rather than creating a hole (`pkg/block/engine/readwrite.go:281`), so honoring it requires no extra work — but the handler must not refuse the flag, and a test must assert that. Rejecting it would violate a MUST on the strength of the same rule that makes `NO_HOLE` invalid on `WRITE`.

So the flag mask differs by command:

| Command | Accepted flags |
|---|---|
| `READ` | `FUA` (ignored) |
| `WRITE` | `FUA` |
| `FLUSH` | `FUA` |
| `TRIM` | `FUA` |
| `WRITE_ZEROES` | `FUA`, `NO_HOLE` |

- [ ] **Step 4: Wire the dispatch**

Add `CmdTrim` → `handleTrim`, `CmdWriteZeroes` → `handleWriteZeroes`. `CmdCache` (5) and `CmdBlockStatus` (7) stay unimplemented and answer `ErrEINVAL` — the spec's value for an unknown command.

- [ ] **Step 5: Land the `Timeouts.Idle` arm Task 4's committed loop is missing**

A fix to already-committed code, not new surface. It belongs here because this task already edits `connection.go` and already runs the package tests.

The loop at `internal/adapter/nbd/connection.go:203` arms the wait for the next request header from `h.timeouts.Read` unconditionally. The plan has mandated since commit `056b547f` that `Idle` bound that wait and `Read` bound finishing a request already started — the split NFS uses (`pkg/adapter/nfs/connection.go:312` / `:338`) and SMB uses (`pkg/adapter/smb/connection.go:230` / `:258`). That plan-side fix landed after Task 4 was committed, so the code never got it and `Idle` is dead: an operator who sets `Idle` below `Read` sees it silently ignored.

Measured with `Timeouts{Read: 5s, Idle: 200ms}` and a silent post-handshake connection: `serve still running after 1.5s`. At the defaults both are 5m, which is why nothing caught it.

Replace the arm with the block the plan specifies in Task 4 Step 3 (plan:1709-1715):

```go
// Idle bounds the wait for a header; Read bounds finishing a request that has
// already started (the payload read in handleWrite). At the defaults both are
// 5m, so the distinction only shows for an operator who separates them -- which
// is exactly when the field's own doc has to be true rather than nearly true.
//
// Idle == 0 means no idle timeout, so it cannot be folded into a min(): the
// guard is an independent arm, the same shape NFS uses at
// pkg/adapter/nfs/connection.go:312.
budget := h.timeouts.Read
if h.timeouts.Idle > 0 {
	budget = h.timeouts.Idle
}
if err := conn.SetReadDeadline(time.Now().Add(budget)); err != nil {
	return err
}
```

**And arm `Read` before the WRITE payload read**, the other half of the split, also missing: `handleWrite` reads `req.Length` bytes through `io.ReadFull` (`internal/adapter/nbd/handlers_write.go:69`) under whatever deadline the header read left behind, so without this `Read` bounds nothing and the payload read inherits the `Idle` budget.

The test that pins it must use `Idle` **shorter** than `Read`, or it passes either way:

```go
func TestConnection_IdleDeadlineClosesASilentConnection(t *testing.T) {
	// Timeouts{Read: 5 * time.Second, Idle: 200 * time.Millisecond}: complete
	// the handshake, then send nothing. serve must return within ~1s. With
	// Idle == Read or Idle > Read this passes whether or not the arm exists --
	// which is why the committed TestConnection_PartialHeaderTimesOut
	// (connection_test.go:696, Idle 5s / Read 200ms) cannot catch it.
}
```

Leave the arm inside `if !h.draining()`, which is already correct: `initiateShutdown` sets a 100 ms deadline on every active connection (`pkg/adapter/base.go:447`) and a fresh `SetReadDeadline` replaces it, so arming the full budget while draining would hold the drain open for it.

- [ ] **Step 6: Run the tests and confirm they pass**

Run: `go test ./internal/adapter/nbd/ -race -v`
Expected: PASS.

- [ ] **Step 7: Add the integration assertions**

```go
func TestIntegration_TrimFreesAndReadsZero(t *testing.T) {
	// Write 1 MiB, Flush, DrainAllUploads, then PunchHole through the handler
	// (h.meta.PunchHole then bs.PunchHole). Assert two things, in this order:
	//
	//   1. the punched range reads back as zeros;
	//   2. the manifest shrank — file.Blocks, read before and after through
	//      rt.GetMetadataService().GetFile(ctx, exp.Handle), has fewer entries
	//      and references fewer bytes.
}

func TestIntegration_WriteZeroesDoesNotMaterializeChunks(t *testing.T) {
	// Write 1 MiB, then WRITE_ZEROES over it. Same two assertions: reads
	// zeros, and the manifest shrank rather than growing.
}
```

**Do NOT assert that `bs.DataExtents` stops reporting the punched range. That assertion is unsatisfiable — it was measured, not reasoned.** Two structural reasons:

1. **The punch writes a record, so the local tier keeps answering "data here".** `engine.Store.PunchHole` zero-overwrites the range through the local append log (`pkg/block/engine/readwrite.go:281`) — that is the mechanism that guarantees reads return zeros regardless of rollup state. The side effect is that the local tier now holds an interval covering the punched range.
2. **`DataExtents` over-reports on purpose.** Its doc says *"The map may over-report data but must never under-report it"* (`pkg/block/engine/dataextents.go:41`), because NFSv4.2 SEEK and READ_PLUS callers must never skip real data. Reporting a hole where a punch happened is exactly the over-report the polarity permits — a client that copies zeros loses nothing.

Measured against the real fixture: 1 MiB written and 256 KiB punched → `DataExtents` still `[[0 1048576]]`; a punch aligned to a real chunk extent over 4 MiB of random data → still `[[0 4194304]]`; while the same punches took `file.Blocks` from **4 to 3** and the referenced bytes from **4194304 to 3095584**, and the punched range read back as zeros.

**So the manifest is the observable.** `metadata.Service.PunchHole` prunes `file.Blocks` and persists the new manifest; that pruning is the reclaim, and `GetFile` is how a test sees it. The engine's own reclaim test asserts the same thing the same way (`pkg/block/engine/api_blockref_test.go:270`, which checks the returned `kept` list and the coordinator's reap calls, never the extent map).

**One more trap in the same step:** `Flush` alone is not enough to populate the manifest. With a durable local store it returns before the remote drain (`pkg/block/engine/flush.go:554`), so `file.Blocks` is still empty and the punch has nothing to prune — a test written that way passes for the wrong reason. Call `bs.DrainAllUploads(ctx)` after `Flush` and assert `len(file.Blocks) > 0` before punching, exactly as Task 4's eviction test does.

- [ ] **Step 8: Verify the WRITE_ZEROES test actually bites**

Change `handleWriteZeroes` to call `bs.WriteAt(ctx, string(exp.PayloadID), nil, make([]byte, req.Length), req.Offset)` instead of the punch pair. Run `go test ./internal/adapter/nbd/ -run 'TestConnection_WriteZeroesThenReadIsZeros|TestIntegration_WriteZeroesDoesNotMaterializeChunks' -v` and confirm BOTH fail: the unit test on `st.punches == 1` / `st.writes == 0`, and the integration test on the manifest growing instead of shrinking. Restore both.

- [ ] **Step 9: Run the affected suites**

Run: `go build ./... && go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/...`
Expected: clean.

**Do not run `make test-unit`** (`go test -race ./...`) and read a timeout as a failure: five packages — `internal/adapter/nfs/v3/handlers`, `internal/adapter/smb/handlers`, `pkg/block/chunker`, `pkg/block/engine`, `pkg/block/middleware/encryption` — hit the 10m deadline on this machine from SQLite contention under `-race`, reproducible on a clean base and unrelated to this work. They were verified environmental during Task 3.

- [ ] **Step 10: Commit**

```bash
git add internal/adapter/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): TRIM and WRITE_ZEROES via PunchHole"
```

---

