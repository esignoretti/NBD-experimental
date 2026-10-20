## Task 5: WRITE, FUA, and FLUSH

**Files:**
- Create: `internal/adapter/nbd/handlers_write.go`, `internal/adapter/nbd/handlers_control.go`
- Modify: `internal/adapter/nbd/connection.go` (dispatch), `internal/adapter/nbd/handlers_read.go` (none)
- Test: `internal/adapter/nbd/connection_test.go`, `internal/adapter/nbd/integration_test.go`

**Interfaces:**
- Consumes: Task 4's `BlockStore` interface, `Handler`, `mapError`; `internal/adapter/common.WriteToBlockStore` (`write_payload.go:54`), `internal/adapter/common.CommitBlockStore` (`:110`) with the `Committer` parameter Task 4 introduced.
- Produces:
  - `func (h *Handler) handleWrite(ctx context.Context, bs BlockStore, exp *Export, req *Request, conn net.Conn) error`
  - `func (h *Handler) handleFlush(ctx context.Context, bs BlockStore, exp *Export, req *Request, conn net.Conn) error`
  - `func (h *Handler) commit(ctx context.Context, bs BlockStore, exp *Export) error` — the shared FUA/FLUSH barrier

**The write path is four steps, and they are the same sequence the NFS handler uses** (`internal/adapter/nfs/v4/handlers/write.go:173`, `:209`, `:218`) — the only difference is that the NFS handler's `common.WriteToBlockStore` wrapper is inlined here, for the reason below:

```
authCtx := h.authContext(ctx, exp)   // per command: it embeds the command's Context
intent, err := h.meta.PrepareWrite(authCtx, exp.Handle, newSize)
bs.WriteAt(ctx, string(intent.PayloadID), nil, data, offset)
_, err = h.meta.CommitWrite(authCtx, intent)
```

**Step 3 is a direct `bs.WriteAt`, not `common.WriteToBlockStore`.** `WriteToBlockStore` takes `*engine.Store` concretely, and the handler holds a `BlockStore`. What `WriteToBlockStore` adds over the raw call is deadline wrapping (`WithRequestDeadline`) and error normalization (`normalizeBlockStoreError`) — so the handler must do both itself, and that is not optional:

```go
ctx, cancel := common.WithRequestDeadline(ctx)
defer cancel()
if _, err := bs.WriteAt(ctx, string(intent.PayloadID), nil, data, req.Offset); err != nil {
    return mapError(err)
}
```

**`mapError` is the single place the wire layer converts an error, and it calls `common.NormalizeBlockStoreError` internally** — so the handler passes the raw error and needs no explicit normalize call. Normalizing first matters: the engine returns raw sentinels as well as `*merrs.StoreError`, and normalization is what turns a raw one into a `*merrs.StoreError` with the right code while leaving an already-wrapped one alone (`if goerrors.As(err, &storeErr) { return err }`). The failure mode of getting this wrong is an error code that silently degrades to `ErrEIO` because `errors.Is` no longer matches through an extra wrap — so a disk-full becomes an I/O error and the client has no way to know its device is full.

**Task 4 already exported it.** `NormalizeBlockStoreError` is at `internal/adapter/common/normalize.go:27` and `mapError` calls it at `internal/adapter/nbd/connection.go:291`. Nothing to do here — recorded so nobody goes looking for a rename that has already happened.

- [ ] **Step 1: Write the failing tests**

The names below are the contract; the bodies are yours to write. They are **not** transcribed because they cannot be: `handleWrite` calls `h.meta.PrepareWrite` and `h.meta.CommitWrite`, and the connection tests' `newTestHandler` passes **`nil`** for `meta` (`internal/adapter/nbd/connection_test.go:151`) — correct for Task 4, where nothing on the read path touched metadata. So every WRITE test that expects `Error == 0` needs a handler with a real metadata service.

**Add a second fixture helper** beside `newTestHandler`, and use it for the write tests:

```go
// newWriteTestHandler builds a handler whose metadata service is real. The
// read-path helper passes nil because READ never touches metadata; WRITE does,
// through PrepareWrite and CommitWrite, so it needs a service whose store is
// the same share the fake block store stands in for.
func newWriteTestHandler(t *testing.T, st BlockStore, exp *Export) *Handler
```

Build it the way `integrationHandler` does (`internal/adapter/nbd/integration_test.go:96`): a `runtime.Runtime` from `testruntime.NewWithShareStore(t)` with one share and one metadata store, `rt.GetMetadataService()` for the service, and `exp.Handle` set to a real volume file's handle — a volume file created under the share's root with `CreateFile` and then sized with `SetFileAttributes`, because `ApplyCreateDefaults` zeroes the size (`pkg/metadata/validation.go:492`). Reuse the fixture that `integration_test.go` already builds if it can be factored into a helper both files call; do not build a second one that drifts.

Tests whose assertions are *rejections* need no metadata (the handler returns before touching it), so they may keep `newTestHandler`. Tests that must reach `Error == 0` on a WRITE cannot.

```go
func TestConnection_WritePersistsBytes(t *testing.T)
func TestConnection_WritePastEndIsENOSPC(t *testing.T)
func TestConnection_WriteZeroLengthIsEINVAL(t *testing.T)
func TestConnection_RejectedWriteKeepsTheStreamSynchronized(t *testing.T)
func TestConnection_RejectedHugeWriteDoesNotAllocate(t *testing.T)
func TestConnection_WriteTruncatedPayloadIsAnError(t *testing.T)
func TestConnection_WriteWithFUAIsDurableBeforeReply(t *testing.T)
func TestConnection_FlushIsAckedWhenDurable(t *testing.T)
func TestConnection_FlushIsEIOWhenNotDurable(t *testing.T)
func TestConnection_FlushWithNonZeroLengthIsEINVAL(t *testing.T)
func TestConnection_FlushOnReadOnlyExportIsPermitted(t *testing.T)
```

What each must assert, in the terms the test file already uses (`sendRequest`, `readReplyHeader`, `readPayload`, `assertNoPayload`, `completeHandshake`, `pipe`):

- **`WritePersistsBytes`** — WRITE 4096 bytes at offset 8192 **with its payload**; `rep.Error == 0`; then a READ of `[8192, 12288)` on the same connection returns those bytes byte-exact. This is the only test that proves a WRITE round-trips, so it must assert the bytes rather than the error code.
- **`WritePastEndIsENOSPC`** — `Offset: exp.Size-4096, Length: 4097`, **payload sent**, expect `ErrENOSPC`. Not `EINVAL`: the spec reserves `ENOSPC` for a write past the end and `EINVAL` for a read or trim. The export size is fixed, so a WRITE past the end is refused, never grown.
- **`WriteZeroLengthIsEINVAL`** — `Length: 0` → `ErrEINVAL`, and **no drain** (there is no payload to consume). Then a valid READ on the same connection must succeed, which is what proves the zero-length path did not consume the next header.
- **`RejectedWriteKeepsTheStreamSynchronized`** — the desync guard, and the highest-value test in the task. Send a WRITE rejected on the range check (`Length: 4096`, offset past the end) **with its 4096-byte payload**, then a valid READ, and expect the READ's reply. Without the drain, the "request header" the server reads next is payload bytes.
- **`RejectedHugeWriteDoesNotAllocate`** — `Length: 1<<30` on a 1 MiB export, **send a little of the payload, then stall**. `ErrENOSPC` must arrive (the range check precedes both the reply and the drain) within `Timeouts.Read`, the process must not grow by 1 GiB, and `st.writes == 0`. Because the reply precedes the drain, the client gets the error it can act on; `Serve` then ends on the drain's read deadline, which is the correct outcome and the test should assert it rather than treat it as a failure. Under the rejected alternative — drain first — this assertion would be unreachable, since a stalled client never completes the drain and so would never see a reply at all.
- **`WriteTruncatedPayloadIsAnError`** — declare `Length: 4096`, send 100 bytes, then close. `Serve` returns an error and never reports success. Distinct from the huge-write case: here the client stopped mid-payload on a request the server had accepted.
- **`WriteWithFUAIsDurableBeforeReply`** — a fake whose `Flush` records whether the reply had already been written. This is the assertion that makes FUA mean something; it is the one Step 8 proves by moving the commit call.
- **`FlushIsAckedWhenDurable`** — `CommitAck: journal` semantics: an fs local store, `Flush` returning `&block.FlushResult{Finalized: false}, nil`. `rep.Error == 0`.
- **`FlushIsEIOWhenNotDurable`** — `requireDurable` set, `Flush` returning `{Finalized: false}, nil`, and the store reporting `localDurable`/`remoteDurable` both false. Expect `ErrEIO`: FLUSH promised durability and could not deliver it, so there is no honest success to send.
- **`FlushWithNonZeroLengthIsEINVAL`** — the spec requires `length` and `offset` to be zero on FLUSH.
- **`FlushOnReadOnlyExportIsPermitted`** — a read-only export still answers FLUSH successfully. Refusing would make a client error a device that has nothing to flush.

- [ ] **Step 2: Run them and confirm they fail**

Run: `go test ./internal/adapter/nbd/ -run 'TestConnection_Write|TestConnection_Flush' -v`
Expected: FAIL.

Add `TestConnection_RejectedWriteKeepsTheStreamSynchronized` to **Review Focus** item 6 if it is not already there — it is the highest-value test in this task, because the failure it catches is silent and remote (a client sees a protocol error minutes later on an unrelated command).

**Delete `TestConnection_WriteIsEINVALBeforeTask5`** (`internal/adapter/nbd/connection_test.go:338`). It pins the dispatcher's `default` branch for a command the adapter advertises but does not yet serve, and after this task WRITE is served — so it asserts the opposite of the intended behaviour and will fail. `TestConnection_UnknownCommandIsEINVAL` and `TestConnection_UnimplementedCommandIsEINVAL` still cover the `default` branch for values the spec does not define. **Update `TestConnection_WriteWithNoHoleFlagIsEINVAL` rather than deleting it** (see Step 3 check 6): it pins a real rule — `NO_HOLE` is not a `WRITE` flag — but it must now send the payload its header declares.

- [ ] **Step 3: Implement `handleWrite`**

Order of checks, and the order is the point:

1. `req.Flags &^ FlagFUA != 0` → `ErrEINVAL` (`NO_HOLE`, `DF`, `REQ_ONE`, `FAST_ZERO` are not applicable to `WRITE`; the spec names `EINVAL` for *"a request with a flag not explicitly documented as applicable to the given request"*).
2. `exp.ReadOnly` → `ErrEPERM`.
3. `req.Length == 0` → `ErrEINVAL`.
4. `req.Offset > math.MaxUint64 - uint64(req.Length)` → `ErrEIO`.
5. `req.Offset + uint64(req.Length) > exp.Size` → `ErrENOSPC` (**not** `EINVAL`). The spec: *"The server SHOULD return `NBD_ENOSPC` if it receives a write request including one or more sectors beyond the size of the device. ... It SHOULD return `NBD_EINVAL` if it receives a read or trim request including one or more sectors beyond the size of the device."* A write past the end means the client's view of the device is too large; a read or trim past the end is a malformed request. Different causes, different codes.
6. **Reply first, then drain the payload — on every error path from here on, except a zero-length one.** The client has already sent `req.Length` bytes — the spec says so unconditionally (*"The client MUST follow the request header with *length* number of bytes"*). If the handler returns without consuming them, those bytes are read as the *next request header* and the connection desynchronizes into garbage. Check 3 already returned for `Length == 0`, so **no drain happens there**; every rejection that reaches this step has a non-zero payload and must discard it.

   **The reply goes out BEFORE the drain, and the two are independent.** The reply travels server→client and the payload travels client→server, so there is no stream-ordering constraint between them — the drain's obligation is to consume exactly `req.Length` bytes so the next header is read from the right offset, and that obligation is unchanged by when the reply is written. Replying first is the better order: a client that declared a huge length and stalled otherwise gets no reply at all, just a TCP close, which its kernel reports as a transport failure rather than the specific error it can act on. A client that has seen the error may also abandon the connection, and the server must not be blocked waiting on bytes that will never come.

   **This changes what the existing flag-rejection test must send.** `TestConnection_WriteWithNoHoleFlagIsEINVAL` (`internal/adapter/nbd/connection_test.go:578`) sent a bare `WRITE` header with `Length: 4096` and no payload, because before this task the dispatcher's `default` branch answered without ever reading one. Once `handleWrite` exists, the flag check returns `ErrEINVAL`, the reply is written, and then the drain blocks waiting for 4096 bytes that never arrive — so the test reads its reply fine but `Serve` is left draining a payload that will never come. **Update it to send its payload** so the drain completes and the connection stays usable. A test that declares bytes it does not send is asserting against a client the spec forbids.

   The drain's deadline is the request budget, re-armed. `SetReadDeadline` was set once before `ReadRequest` and `ReadRequest` read exactly 28 bytes, so what remains is already partly spent — and the drain is where a hostile or stalled client parks. Re-arming from `Timeouts.Read` is what bounds `RejectedHugeWriteDoesNotAllocate`: a client that declares 1 GiB and sends nothing is dropped after one timeout rather than held indefinitely:

   ```go
   // Reject path: the reply goes first, then the payload is consumed. The two
   // travel in opposite directions, so there is no ordering constraint between
   // them — and replying first is what lets a client that stalled mid-payload
   // still learn why, instead of seeing only a closed socket.
   if err := h.writeReply(conn, req.Cookie, nbdErr, nil); err != nil {
       return err
   }
   // Skipping the drain would leave the next header to be read out of payload
   // bytes. Bounded by a deadline so a client that declares a huge length and
   // sends nothing cannot hold the connection open.
   if err := conn.SetReadDeadline(time.Now().Add(h.timeouts.Read)); err != nil {
       return err
   }
   if _, err := io.CopyN(io.Discard, conn, int64(req.Length)); err != nil {
       return err // the stream position is unknown now; the connection must close
   }
   return nil // drained: the connection stays usable for the next command
   ```

   A failed drain returns the error, ending `Serve` and closing the connection — the byte position is unknown, so continuing would read garbage as the next header. A successful drain returns nil and the loop continues, which is what keeps the connection usable for `RejectedWriteKeepsTheStreamSynchronized` and the updated NO_HOLE test.

   `io.CopyN(io.Discard, ...)` rather than `make([]byte, req.Length)`: a client that declares a 4 GiB length must cost the server a bounded buffer, not an allocation. This is the same class as Review Focus 1, one layer down.
7. **Then** read the payload into a buffer sized to the request, using the same bounded shape so a truncated payload costs no allocation either:

   ```go
   data := make([]byte, req.Length)
   if _, err := io.ReadFull(io.LimitReader(conn, int64(req.Length)), data); err != nil {
       return err // the client stopped mid-payload; the connection is unusable
   }
   ```

   A truncated payload returns the read error — the connection is unusable at that point, so returning is correct and the client gets a TCP close rather than a success reply for bytes the server never saw.
8. `authCtx := h.authContext(ctx, exp)`, then `metaSvc.PrepareWrite(authCtx, exp.Handle, req.Offset+uint64(req.Length))` — `metaSvc` is `h.meta`, and `exp.Handle` was resolved once at handshake and has been on the handler ever since. (The auth context is built per command from the command's context, not cached: it embeds `Context`, and a cached one would carry the first command's deadline and cancellation into every later command on the connection.)
9. `bs.WriteAt(ctx, string(intent.PayloadID), nil, data, req.Offset)` on a deadline-wrapped ctx (see above), with the returned `[]block.ChunkRef` discarded — the canonical projection happens at Flush time, not here (`pkg/block/engine/readwrite.go` doc).
10. `h.meta.CommitWrite(authCtx, intent)`.
11. If `req.Flags&FlagFUA != 0`, `h.commit(ctx, bs, exp)` — **before** writing the reply. Spec: *"it MUST NOT send its reply to that command until all write operations... have been completed and persisted to non-volatile storage."*
12. Write the reply with no payload.

Errors from steps 8–11 map through `mapError`. A failure at step 9 or 10 must not write the reply header as success — and note that by then the payload has already been consumed, so the stream stays synchronized.

**`decision:`** the adapter writes through `bs.WriteAt` on a deadline-wrapped context and routes every error through one `mapError`, rather than calling `common.WriteToBlockStore`. That helper takes `*engine.Store` concretely, and widening it to an interface would force a second one for the read path; the deadline and normalization behaviour it provides is two lines and is reproduced here. The commit seam is the one that must be shared, and it is (`common.CommitBlockStore`).

- [ ] **Step 4: Implement `commit` and `handleFlush`**

`commit` calls `common.CommitBlockStore(ctx, bs, exp.PayloadID)` — the same function NFS COMMIT and SMB Flush call, with the same `requireDurable → localDurable || (Finalized && remoteDurable)` rule. This is the reason `BlockStore` carries the three durability accessors. `handleFlush` checks `req.Length == 0 && req.Offset == 0` → `ErrEINVAL` otherwise, then calls `commit`, then replies.

The durability decision is **not reimplemented**: `CommitBlockStore` already encodes `requireDurable → localDurable || (Finalized && remoteDurable)` and returns `ErrNotDurableYet` otherwise (`internal/adapter/common/write_payload.go:110`–`:171`). `mapError` turns that into `ErrEIO`.

**`decision:`** the FLUSH barrier is correct because dispatch is serialized (Task 4). Every write whose reply the client has already seen was handled and replied before this command was read, so "all already-replied writes are durable" follows from the ordering rather than from tracking. If pipelining is ever added, this stops being true and the barrier needs explicit tracking — say that in the comment.

- [ ] **Step 5: Wire the dispatch**

Add `CmdWrite` → `handleWrite` and `CmdFlush` → `handleFlush` to the switch in `connection.go`.

- [ ] **Step 6: Run the tests and confirm they pass**

Run: `go test ./internal/adapter/nbd/ -race -v`
Expected: PASS.

- [ ] **Step 7: Add the integration assertions over the real store**

```go
func TestIntegration_FlushWithCommitAckJournalSucceeds(t *testing.T) {
	// fs local store (from testruntime.New's JournalRoot), requireDurable false.
	// Expect CommitBlockStore to return nil: LocalDurable() is true and
	// pkg/block/engine/flush.go:554 returns after a real fsync.
}

func TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO(t *testing.T) {
	// bs.SetRequireDurableCommit(true) (pkg/block/engine/engine.go:415) with a
	// memory remote. Expect ErrEIO through the handler.
}

func TestIntegration_WriteThenFlushThenReadBackByteExact(t *testing.T) {
	// 128 KiB of pseudorandom data through WRITE+FLUSH, then READ. Byte-exact.
}
```

- [ ] **Step 8: Verify the FUA ordering test actually bites**

In `handleWrite`, move the `h.commit` call to *after* `writeSimpleReply`. Run `go test ./internal/adapter/nbd/ -run TestConnection_WriteWithFUAIsDurableBeforeReply -v` and confirm it fails. Restore.

- [ ] **Step 9: Commit**

```bash
git add internal/adapter/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): WRITE, FUA, and FLUSH over the existing durability seam"
```

---

