## Task 4: Connection loop and READ

**Files:**
- Create: `internal/adapter/nbd/connection.go`, `internal/adapter/nbd/handlers_read.go`
- Modify: `internal/adapter/common/write_payload.go` (`CommitBlockStore` parameter widened to the new `Committer` interface — a type-only change, no behaviour change; every existing call site passes `*engine.Store` and keeps compiling), `internal/adapter/common/normalize.go` (`normalizeBlockStoreError` exported as `NormalizeBlockStoreError`), `internal/adapter/common/{doc.go,read_payload.go,write_payload.go}` (the rename's four call sites: `read_payload.go:60`, `write_payload.go:68`, `:122`, `:171`), `pkg/adapter/nbd/adapter.go` (Step 6 replaces Task 3's `Serve` body and `NewConnection` stub with the real wiring), `pkg/adapter/nbd/config.go` (Step 6 adds `Config.timeouts()`), `pkg/adapter/nbd/healthcheck.go` (Step 7 adds `Degraded`)

**Task 3 must land first**, because both tasks modify `pkg/adapter/nbd/adapter.go`. Task 3 writes the lifecycle, the bind gate and the stubs; this task's Step 6 replaces the two stub bodies. They touch different functions, so the second edit is a substitution rather than a conflict.
- Test: `internal/adapter/nbd/connection_test.go`, `internal/adapter/nbd/integration_test.go`, `internal/adapter/common/write_payload_test.go` (only if one does not already cover `CommitBlockStore`)

**Interfaces:**
- Consumes: Task 2's `Export`/`Exporter`/`Identity`, Task 3's `Config` (timeouts), `Adapter`.
- Produces:
  - `type Timeouts struct { Read, Write, Idle time.Duration }` — declared here, in `connection.go`. `Read` bounds one request header or payload; `Idle` bounds a connection that has completed its handshake but sends nothing, so `Read` must be the *shorter* of the two or a stalled client is held open by the wrong timer. Task 3 stores the operator-facing `TimeoutsConfig` (four fields); Step 6 below adds `Config.timeouts()` to narrow four to these three, dropping `Shutdown` because the base adapter owns the drain deadline.
  - `type StoreResolver func(ctx context.Context, exp *Export) (BlockStore, error)` — a function type rather than a method on `Handler`, so the dispatch tests can inject a fake store. The production implementation wraps `common.ResolveForWrite` (below), so `internal/adapter/nbd` never imports `pkg/controlplane/runtime`.
  - `type BlockStore interface { ReadAt(...); WriteAt(...); PunchHole(...); Flush(...); LocalDurable() bool; RemoteDurable() bool; RequireDurableCommit() bool }` — the four I/O methods plus the three durability accessors, so `commit` can call `common.CommitBlockStore` without the concrete `*engine.Store`. `*engine.Store` satisfies all seven (`pkg/block/engine/readwrite.go:21`, `:87`, `:286`; `pkg/block/engine/flush.go:534`; `pkg/block/engine/engine.go:388`, `:397`, `:406`).
  - `type Handler struct { exporter Exporter; stores StoreResolver; meta *metadata.Service; id Identity; timeouts Timeouts; draining func() bool; degraded atomic.Int64 }`
  - `func NewHandler(exporter Exporter, stores StoreResolver, meta *metadata.Service, id Identity, timeouts Timeouts, draining func() bool) *Handler` — `draining` is a closure over `a.BaseAdapter.Shutdown` from the adapter in Task 3; passing it as a function rather than the base adapter keeps `internal/adapter/nbd` free of any `pkg/adapter` import.
  - `func (h *Handler) serve(ctx context.Context, conn net.Conn) error` — unexported, because it is the test entry point, not the framework's. `type Conn struct { handler *Handler; conn net.Conn }` plus `func NewConn(h *Handler, conn net.Conn) *Conn` and `func (c *Conn) Serve(ctx context.Context)` satisfy `adapter.ConnectionHandler` (`pkg/adapter/base.go:38`, `Serve(ctx)` with no `net.Conn`); the pattern matches `pkg/adapter/smb/connection.go:64`. `Conn.Serve` **must recover panics** — the base adapter's accept loop does not, so an unrecovered panic kills the process rather than the connection (see the code block below).
  - `func (h *Handler) authContext(ctx context.Context, exp *Export) *metadata.AuthContext` — builds the synthetic-identity `AuthContext` from `h.id`, and sets `ShareReadOnly: exp.ReadOnly`. The body is fully determined by those two facts plus the SMB precedent at `internal/adapter/smb/handlers/auth_helper.go:50`:

```go
func (h *Handler) authContext(ctx context.Context, exp *Export) *metadata.AuthContext {
	uid, gid := h.id.UID, h.id.GID
	return &metadata.AuthContext{
		Context:       ctx,
		Identity:      &metadata.Identity{UID: &uid, GID: &gid},
		ShareReadOnly: exp.ReadOnly,
	}
}
```

Built **per command**, never cached: it embeds the command's `Context`, so a cached one would carry the first command's deadline and cancellation into every later command on the connection. `ShareReadOnly` is set **here and not in `RuntimeExporter`** — this is the client-facing ceiling, and the handshake's internal resolution deliberately does not apply it.
  - `type Committer interface { Flush(ctx context.Context, payloadID string) (*block.FlushResult, error); LocalDurable() bool; RemoteDurable() bool; RequireDurableCommit() bool }` in `internal/adapter/common/write_payload.go`, and `CommitBlockStore(ctx context.Context, c Committer, payloadID metadata.PayloadID) error` — the existing function with its parameter widened. `*engine.Store` satisfies it, so the NFS and SMB call sites are untouched.
  - `func mapError(err error) NBDError` (in `connection.go`)

`Handler` deliberately holds no `*runtime.Runtime`: `RuntimeExporter` (Task 2) owns that, and keeping the runtime out of the wire layer is what makes the dispatch tests possible without a control plane.

`Handler` also holds no `net.Conn`, and that is forced by the framework rather than chosen: `adapter.ConnectionHandler` is `interface{ Serve(ctx context.Context) }` (`pkg/adapter/base.go:38`) — no connection parameter — so the `net.Conn` has to arrive some other way. The precedent is `pkg/adapter/smb/connection.go:64 func NewConnection(server *Adapter, conn net.Conn) *Connection`, with `Adapter.NewConnection` at `pkg/adapter/smb/adapter.go:717` returning it. NBD does the same, with a name that does not collide with the framework method:

```go
// Conn is one accepted TCP connection. It exists because
// adapter.ConnectionHandler.Serve takes no net.Conn (pkg/adapter/base.go:38),
// so the socket has to be bound at construction. It holds no protocol state of
// its own: the handshake result and the resolved store live on the stack of
// serve, which is the only thing that can outlive them safely.
type Conn struct {
	handler *Handler
	conn    net.Conn
}

func NewConn(h *Handler, conn net.Conn) *Conn { return &Conn{handler: h, conn: conn} }

// Serve runs one connection and recovers panics.
//
// The recover is not defensive habit: the base adapter's accept loop has none
// (grep recover pkg/adapter/base.go returns nothing) and it invokes this on a
// bare goroutine (pkg/adapter/base.go:373), so an unrecovered panic here takes
// down the whole DittoFS process — every NFS and SMB connection with it. Both
// sibling adapters recover for the same reason and in the same place: NFS at
// pkg/adapter/nfs/connection.go:420 (handleConnectionClose, deferred at :111)
// and SMB at pkg/adapter/smb/connection.go:474. Log at Error, not Debug: a
// panic is unexpected by definition (CLAUDE.md invariant 6), and it is the
// only trace of why the connection died.
func (c *Conn) Serve(ctx context.Context) {
	defer func() {
		if r := recover(); r != nil {
			logger.Error("Panic in NBD connection handler",
				"remote", c.conn.RemoteAddr(), "error", r, "stack", string(debug.Stack()))
		}
	}()
	if err := c.handler.serve(ctx, c.conn); err != nil && ctx.Err() == nil {
		logger.Debug("NBD connection ended", "remote", c.conn.RemoteAddr(), "error", err)
	}
}
```

`Adapter.NewConnection` (`pkg/adapter/nbd/adapter.go`) becomes `return nbd.NewConn(a.handler.Load(), conn)`. The unexported `func (h *Handler) serve(ctx context.Context, conn net.Conn) error` is what the dispatch tests call directly, so they keep the `error` return and never need the wrapper.

**The `handler` field is an `atomic.Pointer[nbd.Handler]`, so `pkg/adapter/nbd/adapter.go` needs `sync/atomic` in its imports.** The pointer is not a stylistic choice: `Serve` stores it on the accept-loop goroutine (`pkg/controlplane/runtime/adapters/service.go:611` starts `adp.Serve` inside `go func()`) while `Healthcheck` loads it from the API handler's goroutine (`internal/controlplane/api/handlers/adapters.go:240` → `Runtime.AdapterChecker` → `adp.Healthcheck`). A plain field is a data race the detector reports on any probe that lands during startup.

**`internal/adapter/nbd/connection.go` needs two imports for that recover block, and neither is obvious:** `github.com/marmos91/dittofs/internal/logger` (a *package*, called as `logger.Error(...)` — the same import `internal/adapter/nfs/v4/handlers/write.go:16` and `internal/adapter/smb/handlers/auth_helper.go:9` use; there is no package-level `logger` variable in `internal/adapter/nbd`) and `runtime/debug` for `debug.Stack()`, as `pkg/adapter/nfs/connection.go:10` does. Without the second, the panic log says what happened but not where, which is the half that matters for a bug that only fires on a client's malformed request.

- [ ] **Step 1: Write the failing connection tests**

Create `internal/adapter/nbd/connection_test.go`. Build a handler over a fake store, then drive it with the Task 1 encoder.

```go
// storeFake is the minimum surface a handler needs. It is deliberately not an
// engine.Store: these tests are about dispatch and error mapping, and a real
// engine would make every failure mode a store failure mode.
type storeFake struct {
	data     map[string][]byte
	readErr  error
	writeErr error
	flushErr error
	reads    int
	writes   int
	flushes  int
}

func (s *storeFake) ReadAt(_ context.Context, id string, buf []byte, off uint64) (int, error) {
	s.reads++
	if s.readErr != nil {
		return 0, s.readErr
	}
	b := s.data[id]
	if off >= uint64(len(b)) {
		for i := range buf {
			buf[i] = 0
		}
		return len(buf), nil
	}
	n := copy(buf, b[off:])
	for i := n; i < len(buf); i++ {
		buf[i] = 0
	}
	return len(buf), nil
}
```

The handler must take an interface, not `*engine.Store`, so this fake is possible — and it must receive the resolver rather than look the store up itself. Define in `connection.go`:

```go
// BlockStore is the engine surface the wire handlers use. An interface rather
// than *engine.Store so the dispatch tests can drive failure modes the engine
// cannot be made to produce on demand. The durability accessors are here
// because the commit rule (common.CommitBlockStore) reads them, and that rule
// must be the same one NFS COMMIT and SMB Flush use.
type BlockStore interface {
	ReadAt(ctx context.Context, payloadID string, data []byte, offset uint64) (int, error)
	WriteAt(ctx context.Context, payloadID string, currentBlocks []block.ChunkRef, data []byte, offset uint64) ([]block.ChunkRef, error)
	PunchHole(ctx context.Context, payloadID string, currentBlocks []block.ChunkRef, offset, length uint64) ([]block.ChunkRef, error)
	Flush(ctx context.Context, payloadID string) (*block.FlushResult, error)
	LocalDurable() bool
	RemoteDurable() bool
	RequireDurableCommit() bool
}
```

`*engine.Store` satisfies all seven. The last three exist only so `commit` can pass the store to `common.CommitBlockStore`, whose parameter this task widens to `common.Committer`. **Do that widening first**, then confirm nothing else broke:

```bash
go build ./... && go test ./internal/adapter/common/ ./internal/adapter/nfs/... ./internal/adapter/smb/...
```

Expected: clean. A type-only widening that breaks a call site means the call site was passing something that is not `*engine.Store`, which is worth knowing before writing new code against it.

`storeFake` must also implement the three durability accessors — returning `true, true, false` for the happy path, so `commit` takes the `localDurable` fast path. The tests that need `ErrNotDurableYet` set them to `false, false, true`. It also needs a `panicOnRead bool` field, so `TestConn_PanicInHandlerDoesNotKillTheProcess` can drive a panic from below the dispatcher; when set, `ReadAt` panics instead of returning. `Handler` takes a `StoreResolver`, and the production resolver is built over **`common.ResolveForWrite`** — do not write a new resolution path.

`internal/adapter/common/resolve.go:15` already declares the seam:

```go
// BlockStoreRegistry is the runtime surface the resolvers need. An interface
// rather than *runtime.Runtime so this package does not import the control
// plane (see the package doc's cycle note).
type BlockStoreRegistry interface {
	GetBlockStoreForHandle(ctx context.Context, handle metadata.FileHandle) (*engine.Store, error)
}

func ResolveForWrite(ctx context.Context, reg BlockStoreRegistry, handle metadata.FileHandle) (*engine.Store, error)
```

So the adapter's resolver is `common.ResolveForWrite(ctx, rt, exp.Handle)` wrapped to return the `BlockStore` interface. `*runtime.Runtime` satisfies `BlockStoreRegistry` (`pkg/controlplane/runtime/runtime.go:830`), which is why the adapter can hold the runtime while the wire layer holds only the function. This is the same call the NFS DEALLOCATE handler makes (`internal/adapter/nfs/v4/handlers/deallocate.go:84`).

**`common.WriteToBlockStore` and `common.CommitBlockStore` take `*engine.Store` concretely, not an interface.** So the wire handlers must NOT call them with a `BlockStore`. The split that keeps both properties — a testable interface for dispatch, and the shared durability rule for production — is:

- The handlers call `bs.WriteAt` / `bs.ReadAt` / `bs.PunchHole` directly through `BlockStore` (they are the same three calls `common` makes internally).
- **`commit` is where the shared rule must be reused, and it is the one place it can be.** `common.CommitBlockStore` is the durability decision (`requireDurable → localDurable || (Finalized && remoteDurable)`, `internal/adapter/common/write_payload.go:110`); reimplementing it in the adapter is exactly the duplication that makes the NBD path drift from NFS COMMIT and SMB Flush. So `BlockStore` needs a way to reach it that does not require the concrete type.

Two acceptable resolutions, in order of preference:

1. **Add a narrow interface for the commit seam in `internal/adapter/common`** — e.g. `type Committer interface { Flush(ctx context.Context, payloadID string) (*block.FlushResult, error); LocalDurable() bool; RemoteDurable() bool; RequireDurableCommit() bool }` and a `CommitBlockStore(ctx, c Committer, payloadID)` overload (rename the existing one's parameter type; `*engine.Store` satisfies it, so every existing call site compiles unchanged). This keeps one implementation of the rule and makes the fake able to drive `ErrNotDurableYet`.
2. If (1) is rejected as too wide a change, have `commit` call `common.CommitBlockStore(ctx, store, exp.PayloadID)` where `store` is the concrete `*engine.Store` the resolver returns — i.e. `StoreResolver` returns both the interface and the concrete store. This works but makes the fake unable to exercise the durability rule, which is the rule most worth exercising.

**Option 1 is what the plan assumes.** It is a two-line change to `internal/adapter/common/write_payload.go` (widen the parameter type), and it is the difference between "the adapter calls the same durability rule as every other protocol" and "the adapter has its own copy of the durability rule."

**Resolve once per connection, not per command.** The handle is fixed for the connection's life, so a per-command resolution would decode a handle and take a lock on every 4 KiB read.

**Use the `pipe(t)` helper for these tests, and do not redefine it.** `connection_test.go` is `package nbd` (internal), the same package as `handshake_test.go`, so the helper declared there (`func pipe(t *testing.T) (server, client net.Conn)`, plan line 411) is already in scope — a second declaration is a redeclaration error. It exists because `net.Pipe` is synchronous and unbuffered: several of these tests read a reply header and assert on it without draining the payload that follows, which with `net.Pipe` leaves the server blocked forever and turns a test that should fail loudly into one that hangs until the Go test timeout (the full reasoning is at plan lines 403-410, where the helper is defined). If you find yourself wanting `net.Pipe` here, read that comment first.

Tests:

```go
func TestConnection_ReadServesBytes(t *testing.T)
func TestConnection_ReadServesZerosForUnwrittenRange(t *testing.T)
func TestConnection_UnknownCommandIsEINVAL(t *testing.T)
func TestConnection_UnimplementedCommandIsEINVAL(t *testing.T)
func TestConnection_WriteIsEINVALBeforeTask5(t *testing.T) // replaced in Task 5
func TestConnection_ErrorsCarryTheCookie(t *testing.T)
func TestConnection_DiscClosesWithoutReply(t *testing.T)
func TestConn_PanicInHandlerDoesNotKillTheProcess(t *testing.T)
func TestConnection_FailedStoreResolutionDegradesAndCounts(t *testing.T)
```

Assertions that matter, spelled out:

- `TestConnection_ReadServesBytes` — write 4096 known bytes into the fake at payload `"p1"`, send `Request{Type: CmdRead, Cookie: 7, Offset: 0, Length: 4096}`, expect a 16-byte reply with `Error == 0` and `Cookie == 7`, followed by exactly 4096 bytes matching the fixture.
- `TestConnection_ReadServesZerosForUnwrittenRange` — read past the fixture's end; expect `Error == 0` and 4096 zero bytes. A genuine hole reads as zeros, not as an error: that is the engine's contract and the adapter must not turn it into `EIO`.
- `TestConnection_UnknownCommandIsEINVAL` — send `Type: 99`; expect `Error == ErrEINVAL` and **no payload**. A reply header for a command whose reply shape is unknown must not be followed by data.
- `TestConnection_UnimplementedCommandIsEINVAL` — send `Type: CmdCache` (5); expect `Error == ErrEINVAL`, not `ErrENOTSUP`. `NBD_ENOTSUP` is reserved by the spec for `WRITE_ZEROES`+`FAST_ZERO`, which we do not advertise; a command we do not implement is an invalid command.
- `TestConnection_ErrorsCarryTheCookie` — with `readErr = errors.New("boom")`, expect `Error == ErrEIO` and the cookie echoed. A dropped cookie makes a client hang forever.
- `TestConnection_DiscClosesWithoutReply` — send `Type: CmdDisc`; expect the server to close the connection with nothing written after the request.

**Review Focus 2 and 3** live here:

```go
// The last byte of the device is addressable. Only offset+length > size is out
// of range.
func TestConnection_ReadAtExactEndIsServed(t *testing.T) {
	// size 8192; request Offset: 4096, Length: 4096 → expect Error == 0 and
	// 4096 bytes, not EINVAL.
}

func TestConnection_ReadPastEndIsEINVAL(t *testing.T) {
	// size 8192; request Offset: 4096, Length: 4097 → expect ErrEINVAL.
}

func TestConnection_ReadZeroLengthIsEINVAL(t *testing.T) {
	// Length: 0 → expect ErrEINVAL and no payload, not a silent empty success
	// and not a dropped connection.
}

func TestConnection_OffsetPlusLengthOverflowIsEIO(t *testing.T) {
	// Offset: math.MaxUint64, Length: 4096 → expect ErrEIO.
	// Not EOVERFLOW: the spec reserves that for READ+DF and payload-less
	// over-max requests, neither of which we support. Not EINVAL either — the
	// request is not out of range, it cannot be interpreted at all.
	// The check must not be `offset+length > size` with a wrapping add.
}

// FUA is valid on every command once SEND_FUA is advertised: the spec says
// "the server MUST accept all commands with this bit set (even by ignoring the
// bit)" and notes "existing clients are known to set this bit on other
// commands". A READ writes nothing, so the bit is ignored and the read succeeds.
// Refusing would break real clients for a guarantee nobody asked for.
func TestConnection_ReadWithFUAFlagSucceeds(t *testing.T) {
	// Flags: FlagFUA, Type: CmdRead → expect Error == 0 and the data.
}

// NO_HOLE is documented for WRITE_ZEROES only, so on WRITE it is a flag "not
// explicitly documented as applicable to the given request" → NBD_EINVAL.
func TestConnection_WriteWithNoHoleFlagIsEINVAL(t *testing.T) {
	// Flags: FlagNoHole, Type: CmdWrite → expect ErrEINVAL.
}

func TestConnection_UnadvertisedFlagIsEINVAL(t *testing.T) {
	// Flags: FlagDF | FlagReqOne | FlagFastZero on a READ → ErrEINVAL each.
}
```

**Review Focus 5:**

```go
// A client that sends a partial header and stalls must not hold the connection
// open forever, and must not make the server allocate for a length it never
// sent.
func TestConnection_PartialHeaderTimesOut(t *testing.T) {
	// Timeouts.Read = 200ms. Write 4 bytes, then block. Expect Serve to return
	// within ~1s with a timeout error, and no reply on the wire.
}
```

**Review Focus 7 and 8:**

```go
// The base adapter's accept loop has no recover (pkg/adapter/base.go:373 runs
// the handler on a bare goroutine), so a panic that escapes here does not kill
// the connection — it kills the process, and with it every NFS and SMB
// connection. Both sibling adapters recover for this reason: NFS at
// pkg/adapter/nfs/connection.go:420, SMB at pkg/adapter/smb/connection.go:474.
//
// A fake store whose ReadAt panics is the cheapest way to reach the recover:
// the panic originates below the dispatcher, exactly where a real one would
// (a nil map, a slice bounds error in a handler, a bad type assertion on a
// store the engine returned).
func TestConn_PanicInHandlerDoesNotKillTheProcess(t *testing.T) {
	// storeFake with panicOnRead set. Drive a full handshake and one READ
	// through Conn.Serve (not handler.serve — the recover lives on Conn).
	// Run Serve on a goroutine and assert it RETURNS within ~2s.
	//
	// The assertion is "it came back", and that is sufficient: if the recover
	// is missing the panic unwinds the goroutine and the test binary dies, so
	// the failure mode is a crashed package rather than a failed assertion.
	// Do NOT wrap the call in a recover() in the test itself — that would make
	// the test pass whether or not Conn.Serve has one.
}

// The store is resolved after the handshake, so a resolver that fails is the
// only way to reach this path deterministically: the window between the
// handshake ACK and the resolution is a few microseconds on the server's own
// goroutine, so no external test can land a share-removal inside it.
func TestConnection_FailedStoreResolutionDegradesAndCounts(t *testing.T) {
	// NewHandler with a StoreResolver returning a sentinel error and a fake
	// exporter whose Resolve succeeds. Drive one full handshake, then assert in
	// this order:
	//   1. serve returned a non-nil error (the connection ends)
	//   2. h.TakeDegraded() == 1
	//   3. a second h.TakeDegraded() == 0
	// The order matters: asserting the consumption first would pass on a
	// counter that was never incremented.
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `go test ./internal/adapter/nbd/ -run TestConnection -v`
Expected: FAIL — `undefined: NewHandler`, `undefined: BlockStore`.

- [ ] **Step 3: Implement `connection.go`**

`serve(ctx, conn)`:
1. `exp, err := ServeHandshake(ctx, conn, h.exporter)`. On error, return it — the handshake already told the client why where the protocol has a way to.
2. Resolve the block store once: `bs, err := h.stores(ctx, exp)`. A failure here is `Degraded` territory (`h.degraded.Add(1)`), not a wire error — the connection ends, because every subsequent command would fail the same way.
3. **Loop, one command at a time.** Set `conn.SetReadDeadline(time.Now().Add(h.timeouts.Read))` before each `ReadRequest`; a deadline error returns a timeout error from `Serve`, closing the connection. On `ErrBadMagic` or any decode error, return it.
4. Switch on `req.Type`. **Every branch must validate flags first**, then bounds, then act — in that order, so a malformed request never reaches the store:

```go
// The order matters: a request that is both malformed and out of range must
// fail on the malformation, because the range check reads the request's own
// fields.
```

5. On `CmdDisc`: return nil without writing a reply. The spec: *"There is no reply to an NBD_CMD_DISC."* Values of length and offset MUST be zero — reject nonzero with `ErrEINVAL` rather than closing.

Set `conn.SetWriteDeadline(time.Now().Add(h.timeouts.Write))` before every reply write.

**Shutdown:** when draining, answer new commands with `ErrESHUTDOWN` rather than serving them. The spec's §4.6 rationale: it is the protocol's "reconnect" signal, which is what the kernel's dead-connection timeout waits for. The draining state is **read from the base adapter, not owned by the handler**: `BaseAdapter` exposes `Shutdown chan struct{}` (`pkg/adapter/base.go:105`), closed by `initiateShutdown` (`:413`), so `Adapter.Serve` passes a closure and there is no second flag to keep in sync:

```go
draining := func() bool {
	select {
	case <-a.BaseAdapter.Shutdown:
		return true
	default:
		return false
	}
}
```

**The handler's own read deadline fights the base adapter's shutdown, and the handler has to lose that fight deliberately.** `initiateShutdown` calls `interruptBlockingReads` (`pkg/adapter/base.go:447`), which sets a **100 ms** read deadline on every active connection to unblock pending reads. But the handler sets its own deadline before each read (`SetReadDeadline(time.Now().Add(h.timeouts.Read))` — 5 minutes by default), and a `SetReadDeadline` **replaces** the previous one. So a handler that finishes a command after shutdown began will overwrite the 100 ms interruption with a fresh 5-minute deadline, and `waitForDrain` will sit until `ShutdownTimeout` (30 s) expires and force-close.

The loop must therefore check `h.draining()` **before** arming its own deadline, and when draining:

- **Do not** arm the long deadline. Arm a short one (or none, leaving the base adapter's 100 ms in place), so the read unblocks promptly and the connection closes inside the drain window.
- If a command *does* arrive in that window, answer it `ErrESHUTDOWN` — that is the polite path the spec wants, and it is reachable only because the client had already written the request.
- **Do not** close the connection on the first draining observation. Closing immediately is what makes the client's kernel error the device instead of pausing it; the base adapter's `ShutdownTimeout` is the backstop, and `waitForDrain` force-closes when it expires.

`ErrESHUTDOWN` on drain is therefore **best-effort by construction**, not a guarantee: an idle connection is closed by the 100 ms deadline long before any command arrives. That is acceptable because a TCP close is exactly what the kernel's `dead_conn_timeout` is watching for (§6.1) — `ESHUTDOWN` is the fast, polite version of the same signal, not a different mechanism.

- [ ] **Step 4: Implement `handlers_read.go`**

```go
func (h *Handler) handleRead(ctx context.Context, bs BlockStore, exp *Export, req *Request, conn net.Conn) error
```

Checks in order: overflow → `ErrEIO`; `req.Length == 0` → `ErrEINVAL`; `req.Offset+uint64(req.Length) > exp.Size` → `ErrEINVAL`; then any flag other than `FlagFUA` → `ErrEINVAL`.

**Flag handling on READ:** `FlagFUA` is *accepted and ignored* — the spec requires the server to accept it on all commands once `SEND_FUA` is advertised (*"the server MUST accept all commands with this bit set (even by ignoring the bit)"*), and a read writes nothing, so there is nothing to persist. Every other flag (`NO_HOLE`, `DF`, `REQ_ONE`, `FAST_ZERO`) is `ErrEINVAL`.

**Overflow is `ErrEIO`, not `ErrEOVERFLOW`.** The spec: *"The server SHOULD NOT return `NBD_EOVERFLOW` except as documented in response to `NBD_CMD_READ` when `NBD_CMD_FLAG_DF` is supported, or when a command without payload requests a length larger than an advertised maximum payload length."* We advertise neither, so the value is out of bounds for us to send.

Then `buf := make([]byte, req.Length)`, `bs.ReadAt(ctx, string(exp.PayloadID), buf, req.Offset)`, and on error `writeSimpleReply(conn, req.Cookie, mapError(err))` with no payload.

**Wrap the context with `common.WithRequestDeadline(ctx)` (`internal/adapter/common/deadline.go:32`) before `ReadAt`.** It returns `(ctx, cancel)`, so `defer cancel()`. Two properties matter: it is a **no-op when the context already carries a deadline** (`if _, ok := ctx.Deadline(); ok { return ctx, func() {} }`), so it is free to call per command; and its own doc says a handler looping over choke points *"takes one deadline up front with this function and passes it down, so the loop shares one budget instead of starting a new one per call."* — for NBD the unit is **one command**, not one connection, because a connection is long-lived and a connection-wide budget would expire a boot disk mid-boot. So call it per command and let `cancel` run at the end of that command.

Every store call in this adapter (read, write, punch, flush) goes through it; that is the same wrapper `ReadFromBlockStore` and `WriteToBlockStore` apply. Skipping it means a hung remote fetch blocks the connection loop past every configured timeout.

**Do not** use `common.ReadFromBlockStore` (`internal/adapter/common/read_payload.go:42`): it takes `*engine.Store` concretely and returns a *pooled* buffer whose `Release()` the caller must call. A pooled buffer that outlives the reply write is a data race waiting to happen, and the handler has no pool. Allocate per-command and let the GC have it.

On success: write the reply header with `Error: 0` and `Cookie: req.Cookie`, then `buf`.

**Do not** allocate `buf` before the bounds checks. A client that sends `Length: 0xffffffff` with a 1 GiB export must get `ErrEINVAL` without a 4 GiB allocation.

- [ ] **Step 5: Implement `mapError`**

```go
func mapError(err error) NBDError
```

**`mapError` must call `common.NormalizeBlockStoreError` (`internal/adapter/common/normalize.go:25`, exported by Task 4) before classifying**, or the `merrs` cases below will not match: the engine returns raw sentinels and `merrs.StoreError` values, and normalization is what turns the raw ones into a `*merrs.StoreError` with the right code while leaving an already-wrapped one alone (`if goerrors.As(err, &storeErr) { return err }`). **Do not** hand-roll a classifier that skips normalization: it will silently report `ErrEIO` for a disk-full condition, which is exactly the class of bug this design exists to prevent.

- `nil` → 0
- `engine.ErrStoreClosed` → `ErrEIO` (the store is gone; the client should error the device, and §4.6 maps this to EIO rather than ESHUTDOWN because ESHUTDOWN means "reconnect" and a closed store is not a restart)
- `common.ErrNotDurableYet` → `ErrEIO`. `internal/adapter/nbd` importing `internal/adapter/common` is safe and verified: `go list -deps ./internal/adapter/common` includes nothing from `internal/adapter/nbd`, so there is no cycle. (This is the same direction the NFS and SMB handlers import in.)
- `merrs.ErrNoSpace` → `ErrENOSPC`
- `merrs.ErrPermissionDenied` → `ErrEPERM`
- `context.Canceled`, `context.DeadlineExceeded` → `ErrESHUTDOWN`
- default → `ErrEIO`

**Order matters where the causes overlap.** `engine.ErrStoreClosed` and the `merrs` pair are tested before the `context` pair, because a store failure wrapped in a cancellation should report the store failure — the operator needs to know the store is gone, not that a context expired. `errors.Is` walks the whole wrap chain, so a single `err` can match several cases; the first match wins.

`mapError` never returns `ErrEOVERFLOW` or `ErrENOTSUP`; both are unreachable by design (see the Global Constraints entry). If a future handler wants one, it must also add the advertisement that makes it legal to send.

Use `errors.Is` throughout, and put the original error in a log line at Debug for expected classes and Error otherwise, per CLAUDE.md invariant 6.

- [ ] **Step 6: Wire the adapter to the wire layer**

This step exists here, not in Task 3, because it is the first point at which every symbol it names exists: `nbd.Handler`, `nbd.NewHandler`, `nbd.BlockStore`, `nbd.StoreResolver`, and `nbd.Timeouts` are all declared by this task. Task 3 deliberately left `Serve`'s body and `NewConnection` as stubs so that its own commit compiles; this step replaces both.

The adapter is the only place that knows about both halves — the runtime and the wire layer — so this is where the two seams meet. It must import `internal/adapter/nbd`, which is legal (both are in-repo and `internal/adapter/nbd` does not import `pkg/adapter/nbd`, so there is no cycle).

```go
type Adapter struct {
	*adapter.BaseAdapter
	config  Config
	runtime *runtime.Runtime
	// handler is built once in Serve, before the listener exists, and read by
	// NewConnection and Healthcheck — both of which run on other goroutines.
	// An atomic pointer, not a plain field: Serve runs in its own goroutine
	// (pkg/controlplane/runtime/adapters/service.go:611) while Healthcheck is
	// reached from an HTTP handler (internal/controlplane/api/handlers/adapters.go:240)
	// via Runtime.AdapterChecker, so a plain field is a data race the race
	// detector reports on any probe that lands during startup.
	handler atomic.Pointer[nbd.Handler]
}

func (a *Adapter) SetRuntime(rt any) {
	r, ok := rt.(*runtime.Runtime)
	if !ok {
		panic(fmt.Sprintf("nbd: SetRuntime wants *runtime.Runtime, got %T", rt))
	}
	// The base adapter keeps its own reference for the shared accept loop,
	// which reaches it for connection metrics (pkg/adapter/base.go:359, :387).
	a.BaseAdapter.SetRuntime(rt)
	a.runtime = r
}

func (a *Adapter) Serve(ctx context.Context) error {
	if netutil.IsNonLoopbackHost(a.config.BindAddress) && !a.config.AllowRemoteBind {
		return fmt.Errorf("nbd: refusing to bind %q without allow_remote_bind; NBD has no authentication beyond optional TLS, so a remotely reachable export hands every byte of the volume to anyone who can reach the port", a.config.BindAddress)
	}
	if a.runtime == nil {
		return errors.New("nbd: SetRuntime was not called before Serve")
	}
	id := nbd.Identity{UID: a.config.UID, GID: a.config.GID}
	exporter := nbd.NewRuntimeExporter(a.runtime, id)
	resolver := func(ctx context.Context, exp *nbd.Export) (nbd.BlockStore, error) {
		return common.ResolveForWrite(ctx, a.runtime, exp.Handle)
	}
	draining := func() bool {
		select {
		case <-a.BaseAdapter.Shutdown:
			return true
		default:
			return false
		}
	}
	a.handler.Store(nbd.NewHandler(exporter, resolver, a.runtime.GetMetadataService(),
		id, a.config.timeouts(), draining))
	return a.ServeWithFactory(ctx, a, nil, nil)
}

func (a *Adapter) NewConnection(conn net.Conn) adapter.ConnectionHandler {
	return nbd.NewConn(a.handler.Load(), conn)
}
```

Also add to `pkg/adapter/nbd/config.go` the accessor this step calls:

```go
// timeouts narrows the adapter's four configured durations to the three the
// wire layer needs. Shutdown is deliberately not here: the base adapter owns
// the drain deadline, and the handler's job during drain is to answer
// ErrESHUTDOWN, not to run its own timer.
func (c *Config) timeouts() nbd.Timeouts {
	return nbd.Timeouts{Read: c.Timeouts.Read, Write: c.Timeouts.Write, Idle: c.Timeouts.Idle}
}
```

Five things this shape gets right, and one it makes explicit:

- **`NewConnection` returns a small per-connection wrapper, not the shared handler.** `adapter.ConnectionHandler.Serve` takes only a context (`pkg/adapter/base.go:38`), so the socket must be bound at construction — `return nbd.NewConn(a.handler.Load(), conn)`, the same shape as `pkg/adapter/smb/adapter.go:717`. The handler itself is shared and stateless; `Conn` carries nothing but the socket and the handler pointer, so there is no per-connection protocol state to leak.
- **The resolver returns `*engine.Store` as `nbd.BlockStore` implicitly** — `common.ResolveForWrite` returns the concrete type, which satisfies the interface. No adapter.
- **`Serve` builds the handler once, before the listener exists**, so a `nil` runtime or a bad config fails as a startup error rather than as a failed first connection.
- **The bind gate runs first, before the runtime is consulted**, which is what `TestServe_RefusesRemoteBindEvenWithNoRuntime` in Task 3 pins. Keep that ordering when you move the body.
- **`SetRuntime` panics on the wrong type**, matching the documented contract of `pkg/adapter/base.go:185` (*"must be a `*runtime.Runtime` or it panics"*) and the constructor-panics-on-invalid-config precedent at `pkg/adapter/smb/adapter.go:176`.
- **`preAccept` stays `nil`** — there is nothing checkable at the TCP layer for a protocol that names its export inside the handshake.

- [ ] **Step 7: Add `Degraded` to the healthcheck**

Add a counter the connection path increments on a failed export resolution or block-store resolution, and have `Healthcheck` report `StatusDegraded` when it is nonzero since the last probe. Message: `"NBD adapter is serving but N export resolutions have failed"`. Keep the probe cheap — read the counter and reset it, no I/O.

The counter lives on `Handler` (`h.degraded`), but `Healthcheck` is on `Adapter` in the other package, so the adapter needs a read-and-reset accessor: `func (h *Handler) TakeDegraded() int64` using `h.degraded.Swap(0)`. `Swap` rather than `Load` then `Store` — two steps let a concurrent increment vanish between them. `Adapter.Healthcheck` calls it only when the handler exists (before `Serve` there is nothing to ask):

```go
func (a *Adapter) Healthcheck(ctx context.Context) health.Report {
	if !a.config.Enabled {
		return health.Report{Status: health.StatusDisabled, Message: "NBD adapter is disabled in config", CheckedAt: time.Now().UTC()}
	}
	if h := a.handler.Load(); h != nil {
		if n := h.TakeDegraded(); n > 0 {
			return health.Report{Status: health.StatusDegraded, Message: fmt.Sprintf("NBD adapter is serving but %d export resolutions have failed", n), CheckedAt: time.Now().UTC()}
		}
	}
	return a.BaseAdapter.Healthcheck(ctx)
}
```

The `Disabled` branch and the final delegation are the same shape as `pkg/adapter/nfs/healthcheck.go`; only the `Degraded` branch is new, and it must come **before** the delegation because `BaseAdapter.Healthcheck` has no way to know about a counter it does not own. `TakeDegraded` is called on every probe, so the count is "since the last probe" — say so in the field's doc, because a reader will otherwise assume it is cumulative.

**The `handler` field must be an `atomic.Pointer[nbd.Handler]`** (Step 6), and this is the step that proves it: `Serve` stores it on the accept-loop goroutine while `Healthcheck` loads it from the API handler's goroutine. A plain field races — verified with a probe reproducing this exact shape under `-race`, which reported `WARNING: DATA RACE` between the `Serve` write and the `Healthcheck` read. `NewConnection` also loads it, and that runs on the accept goroutine, so the atomic is what makes the read legal there too.

**Add a test for the `Degraded` branch, and put it where it can actually be written.** The counter has two halves — an increment on a failed store resolution (`connection.go`) and a read-and-reset on a probe (`pkg/adapter/nbd/healthcheck.go`) — and neither is exercised by the two existing healthcheck tests, which only cover the `Disabled` and pre-`Serve` paths.

The increment half is tested in **`internal/adapter/nbd/connection_test.go`** (`package nbd`), not in the external healthcheck test, and the reason is structural rather than stylistic: the increment happens in `serve` **between** `ServeHandshake` and the store resolution (`Step 3` items 1 and 2), and that window is not reachable from outside — a client sees the handshake ACK, and the resolution runs microseconds later on the server goroutine, so no external test can land a share-removal inside it. The internal test does not need to: it injects a `StoreResolver` that fails directly.

```go
// The store resolution happens after the handshake, so a resolver that fails
// is the only way to reach this path without racing the server's own goroutine.
func TestConnection_FailedStoreResolutionDegradesAndCounts(t *testing.T) {
	// NewHandler with a StoreResolver returning a sentinel error, and a fake
	// exporter whose Resolve succeeds. Drive one full handshake, then assert:
	//   - serve returned an error (the connection ends)
	//   - h.TakeDegraded() == 1
	//   - a SECOND call to h.TakeDegraded() == 0 (the probe consumes the count)
	// Assert in that order, so the test cannot pass by the counter never having
	// been incremented at all.
}
```

The probe half is tested in **`pkg/adapter/nbd/healthcheck_test.go`** (`package nbd_test`), which reaches the counter only through the exported surface — `Adapter.Healthcheck`. Do **not** add a `Handler()` accessor to `Adapter` to reach in: it was considered and rejected as dead surface, since nothing in Tasks 4–8 needs it, and the increment half is already covered internally. Drive it through the real path instead:

```go
// The Degraded branch needs the handler to exist, so this test starts Serve.
// It does not need a failed resolution: removing the share after the handshake
// is not reachable from here (see connection_test.go's note), so the count is
// produced by the only external lever there is — resolving an export that does
// not exist, which is a handshake failure rather than a store failure and does
// NOT increment. That is the point: this test pins the OBSERVABLE contract,
// that a probe with a zero count reports healthy, and leaves the increment to
// the internal test.
func TestNBDAdapter_Healthcheck_NotDegradedWithoutFailures(t *testing.T) {
	// New(config), SetRuntime, start Serve on a cancellable ctx and wait for
	// ListenerReady. Probe twice; expect neither probe to be StatusDegraded.
	// A probe must not invent degradation, and TakeDegraded on a fresh handler
	// must not block or panic.
}
```

- [ ] **Step 8: Write the integration test**

Create `internal/adapter/nbd/integration_test.go`. This is the first test over a **real** `engine.Store`, and it is where the durability contract gets pinned.

```go
func newIntegrationFixture(t *testing.T) (*runtime.Runtime, string, *engine.Store)
```

Use `testruntime.NewWithShareStore(t)` (`internal/adapter/testruntime/runtime.go:53`), register a metadata store, `AddShare` with `BlockStoreID`, create `volumes/boot`, set its size, and return the runtime, the export name, and `rt.GetBlockStoreForShare(share)`. The journal root comes from `testruntime.New`, so the local tier is an fs store — which is what makes `LocalDurable()` true and the FLUSH assertions below meaningful.

**The fixture must pass `Enabled: true` in its `ShareConfig`.** `AddShare` copies the flag verbatim (`pkg/controlplane/runtime/shares/lifecycle.go:548`) and nothing defaults it, so a share built without it is disabled — and the exporter now refuses to resolve a disabled share (`internal/adapter/nbd/volume.go`). Every handler test in this task resolves its export through that gate, so omitting the field fails the whole task with a mystery `ErrUnknownExport` that names nothing about the real cause. Production is unaffected: the DB column defaults true (`pkg/controlplane/models/share.go:41`, `gorm:"default:true"`) and `init.go:314` propagates it, so this is a fixture-only trap.

```go
func TestIntegration_ReadAfterWriteOverRealStore(t *testing.T) {
	// 64 KiB of deterministic non-zero bytes written through the handler's
	// WRITE path (or directly via bs.WriteAt while Task 5 is pending), then
	// read back through the handler's READ path; compare byte-exact.
}

func TestIntegration_ReadEvictedButManifestCoveredRange(t *testing.T) {
	// Write, then make the bytes evictable, then evict, then READ. Expect real
	// bytes, not zeros and not an error: this is the hydrate path
	// (pkg/block/engine/read_internal.go:56).
	//
	// The eviction step is not "write then Flush". Verified with a probe against
	// this exact fixture: after WriteAt + Flush, DrainLocalSynced evicts NOTHING
	// (SegmentsEvicted=0), because the share's journal is durable
	// (pkg/block/journal/store.go:365 sets durable true by default) and
	// RequireDurableCommit is false, so Flush returns early at
	// pkg/block/engine/flush.go:554 with {Finalized: false} and never drains the
	// remote. The bytes stay dirty, and journal.Evict skips any segment holding
	// a dirty record (pkg/block/journal/evict.go:36) — so the range is still
	// local, the read is a warm local read, and the test passes without ever
	// exercising the hydrate path it exists to prove.
	//
	// DrainAllUploads is what makes the bytes evictable: it force-flushes every
	// dirty range to the remote and waits for the uploads to settle. Probe
	// result with it in place: SegmentsEvicted=1, BytesFreed=65676.
	//
	// So the sequence is: WriteAt, DrainAllUploads, DrainLocalSynced, and ASSERT
	// the eviction actually happened before reading:
	//
	//	ev, err := bs.DrainLocalSynced(ctx)
	//	if err != nil { t.Fatal(err) }
	//	if ev.SegmentsEvicted == 0 {
	//		t.Fatalf("nothing was evicted, so this test cannot exercise the hydrate path: %+v", ev)
	//	}
	//
	// That assertion is the whole test's reason for existing. Without it a
	// fixture change that keeps the bytes local turns this into a test that
	// passes while proving nothing — which is worse than no test, because it
	// reads as coverage.
	//
	// The remote is the fixture's `memory` block store (remotememory), whose
	// durability type-default is FALSE (pkg/block/remote/memory/store.go:51-53),
	// so RemoteDurable() reports false and a require-durable commit could not be
	// satisfied here. That is fine for this test: DrainAllUploads does not read
	// the durability flag, it drains the uploads.
}

func TestIntegration_ReadNeverWrittenRangeIsZeros(t *testing.T) {
	// A range inside the export with no write ever. Expect 4096 zeros and
	// Error == 0.
}
```

**Do not expect the eviction test to be satisfied by `Flush` alone, and do not "fix" it by setting `RequireDurableCommit` to true.** The early return at `pkg/block/engine/flush.go:554` is deliberate — its comment records that the synchronous path *"turned every FILE_SYNC/DATA_SYNC WRITE into an inline S3 PutObject the reply waited on: multi-second per-op stalls at ~3% CPU"* — and flipping the flag to make a test's setup convenient would both change the durability contract under test and slow every write. `DrainAllUploads` is the supported way to make bytes evictable on demand.

- [ ] **Step 9: Run the tests and confirm they pass**

Run: `go test ./internal/adapter/nbd/ -race -v`
Expected: PASS.

- [ ] **Step 10: Verify the bounds checks actually bite**

Comment out the overflow check in `handleRead`, run `go test ./internal/adapter/nbd/ -run TestConnection_OffsetPlusLengthOverflowIsEIO -v`, and confirm it fails on the assertion. Restore. Repeat for the exact-end check: change `>` to `>=` and confirm `TestConnection_ReadAtExactEndIsServed` fails.

- [ ] **Step 11: Commit**

```bash
git add internal/adapter/nbd/ pkg/adapter/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): connection loop and READ handler"
```

---

