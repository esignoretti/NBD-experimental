## Task 2: Handshake and export resolution

**Files:**
- Create: `internal/adapter/nbd/handshake.go`, `internal/adapter/nbd/reply.go`, `internal/adapter/nbd/volume.go`
- Test: `internal/adapter/nbd/handshake_test.go`, `internal/adapter/nbd/volume_test.go`

**Interfaces:**
- Consumes: Task 1's constants, `OptionHeader`, `OptionReplyHeader`, `TransmissionFlags`.
- Produces:
  - `type Export struct { Name string; Size uint64; PayloadID metadata.PayloadID; Handle metadata.FileHandle; ReadOnly bool }` — `Handle` is what the write path needs for `PrepareWrite`/`CommitWrite`; it is resolved once at handshake and held for the connection's life.
  - `type Exporter interface { Resolve(ctx context.Context, name string) (*Export, error); List(ctx context.Context) ([]string, error) }`
  - `var ErrUnknownExport = errors.New("nbd: unknown export")`
  - `type Identity struct { UID, GID uint32 }` — the configured synthetic identity (spec §4.7). Declared here, not in Task 4, because `NewRuntimeExporter` below takes one: the handshake needs it to build the `AuthContext` that `metadata.Service.Lookup` requires, so this task's own commit cannot compile without it. An earlier draft declared it in Task 4 and left this task uncompilable until Task 4 landed.
  - `func ServeHandshake(ctx context.Context, conn net.Conn, ex Exporter) (*Export, error)` — writes the greeting, haggles options, returns the resolved export with the transmission phase ready to begin. Returns `ErrUnknownExport`-wrapped errors only through the wire; a client that names no valid export ends with the connection closed by this function.
  - `func (e *Export) TransmissionFlags() TransmissionFlags`
  - `type RuntimeExporter struct { rt *runtime.Runtime; id Identity; volumesDir string }`
  - `func NewRuntimeExporter(rt *runtime.Runtime, id Identity) *RuntimeExporter` — `id` is needed because `Lookup` takes an `*AuthContext` and the handshake has no client identity; see Step 5.
  - `func (r *RuntimeExporter) Resolve(ctx context.Context, name string) (*Export, error)`
  - `func (r *RuntimeExporter) List(ctx context.Context) ([]string, error)`
  - `func writeOptionReply(w io.Writer, opt Option, typ OptionReplyType, data []byte) error` (in `reply.go`)
  - `func writeSimpleReply(w io.Writer, cookie uint64, err NBDError) error` (in `reply.go`)

**Export-name syntax:** `<share>/<volume>`, e.g. `/export/boot`. `volumesDir` is the fixed string `"volumes"`. The file lives at `<share>/volumes/<volume>`, and the share part is resolved with `rt.GetRootHandle(share)` then `metadata.Service.Lookup` down the path.

- [ ] **Step 1: Write the failing handshake tests**

Create `internal/adapter/nbd/handshake_test.go`. Drive both sides over a
**loopback TCP socket**, via the `pipe(t)` helper the block below defines — **not**
`net.Pipe()`. `net.Pipe` is synchronous and unbuffered: a server `Write` blocks until the
reader has consumed every byte, so a test that reads a reply header and stops before the
payload leaves the server blocked and the test hangs until the Go test timeout instead of
failing. Several tests below do exactly that. A loopback socket buffers in the kernel, so
the server finishes its reply whether or not the test reads it.

```go
package nbd

import (
	"bytes"
	"context"
	"encoding/binary"
	"errors"
	"io"
	"net"
	"testing"
	"time"
)

type fakeExporter struct {
	exports map[string]*Export
}

func (f *fakeExporter) Resolve(_ context.Context, name string) (*Export, error) {
	if e, ok := f.exports[name]; ok {
		return e, nil
	}
	return nil, ErrUnknownExport
}

func (f *fakeExporter) List(context.Context) ([]string, error) {
	names := make([]string, 0, len(f.exports))
	for n := range f.exports {
		names = append(names, n)
	}
	return names, nil
}

func newFakeExporter() *fakeExporter {
	return &fakeExporter{exports: map[string]*Export{
		"/export/boot": {Name: "/export/boot", Size: 1 << 30, PayloadID: "payload-1"},
	}}
}

// pipe returns a connected pair over a loopback TCP socket.
//
// net.Pipe would be simpler and is wrong here: it is synchronous and
// unbuffered, so a server Write blocks until the reader has consumed every
// byte. These tests deliberately read a reply header and assert on it without
// draining the payload that follows, which leaves the server blocked forever
// and turns a test that should fail loudly into one that hangs until the Go
// test timeout. A loopback socket buffers in the kernel, so a server can
// finish a reply the test has not read yet.
func pipe(t *testing.T) (server, client net.Conn) {
	t.Helper()
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		t.Fatalf("listen: %v", err)
	}
	defer func() { _ = ln.Close() }()

	type accepted struct {
		conn net.Conn
		err  error
	}
	ch := make(chan accepted, 1)
	go func() {
		c, err := ln.Accept()
		ch <- accepted{c, err}
	}()

	client, err = net.Dial("tcp", ln.Addr().String())
	if err != nil {
		t.Fatalf("dial: %v", err)
	}
	a := <-ch
	if a.err != nil {
		t.Fatalf("accept: %v", a.err)
	}
	t.Cleanup(func() {
		_ = client.Close()
		_ = a.conn.Close()
	})
	return a.conn, client
}

// readGreeting consumes NBDMAGIC, IHAVEOPT and the 16-bit handshake flags, and
// returns the flags. Every client does this before anything else.
func readGreeting(t *testing.T, r io.Reader) uint16 {
	t.Helper()
	var buf [18]byte
	if _, err := io.ReadFull(r, buf[:]); err != nil {
		t.Fatalf("read greeting: %v", err)
	}
	if got := binary.BigEndian.Uint64(buf[0:8]); got != nbdMagic {
		t.Fatalf("greeting magic = %#x, want %#x", got, uint64(nbdMagic))
	}
	if got := binary.BigEndian.Uint64(buf[8:16]); got != ihaveoptMagic {
		t.Fatalf("IHAVEOPT = %#x, want %#x", got, uint64(ihaveoptMagic))
	}
	return binary.BigEndian.Uint16(buf[16:18])
}

func sendClientFlags(t *testing.T, w io.Writer, flags uint32) {
	t.Helper()
	var buf [4]byte
	binary.BigEndian.PutUint32(buf[:], flags)
	if _, err := w.Write(buf[:]); err != nil {
		t.Fatalf("write client flags: %v", err)
	}
}

func sendOption(t *testing.T, w io.Writer, opt Option, data []byte) {
	t.Helper()
	h := &OptionHeader{Magic: ihaveoptMagic, Option: opt, Length: uint32(len(data))}
	if err := h.Encode(w); err != nil {
		t.Fatalf("encode option header: %v", err)
	}
	if len(data) > 0 {
		if _, err := w.Write(data); err != nil {
			t.Fatalf("write option data: %v", err)
		}
	}
}

// goPayload builds an NBD_OPT_GO / NBD_OPT_INFO payload: name length, name,
// 16-bit count of information requests, then the requests.
func goPayload(name string, infos ...uint16) []byte {
	var buf bytes.Buffer
	var n [4]byte
	binary.BigEndian.PutUint32(n[:], uint32(len(name)))
	buf.Write(n[:])
	buf.WriteString(name)
	var c [2]byte
	binary.BigEndian.PutUint16(c[:], uint16(len(infos)))
	buf.Write(c[:])
	for _, i := range infos {
		var b [2]byte
		binary.BigEndian.PutUint16(b[:], i)
		buf.Write(b[:])
	}
	return buf.Bytes()
}

func TestHandshake_GreetingAdvertisesFixedNewstyle(t *testing.T) {
	server, client := pipe(t)

	done := make(chan error, 1)
	go func() {
		_, err := ServeHandshake(context.Background(), server, newFakeExporter())
		done <- err
	}()

	flags := readGreeting(t, client)
	if flags&1 == 0 {
		t.Fatalf("handshake flags %#x do not set NBD_FLAG_FIXED_NEWSTYLE (bit 0)", flags)
	}
	// The server must not proceed before reading client flags.
	_ = client.Close()
	select {
	case <-done:
	case <-time.After(2 * time.Second):
		t.Fatal("ServeHandshake did not return after the client closed")
	}
}

func TestHandshake_GoReturnsExportInfoThenAck(t *testing.T) {
	server, client := pipe(t)

	got := make(chan *Export, 1)
	go func() {
		exp, err := ServeHandshake(context.Background(), server, newFakeExporter())
		if err != nil {
			got <- nil
			return
		}
		got <- exp
	}()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)                                 // NBD_FLAG_C_FIXED_NEWSTYLE
	sendOption(t, client, OptGO, goPayload("/export/boot", 0, 3)) // NBD_INFO_EXPORT, NBD_INFO_BLOCK_SIZE

	// NBD_INFO_EXPORT is mandatory before a successful ACK.
	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read first option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepInfo {
		t.Fatalf("first reply type = %#x, want NBD_REP_INFO (%#x)", typ, uint32(RepInfo))
	}
	data := make([]byte, binary.BigEndian.Uint32(hdr[16:20]))
	if _, err := io.ReadFull(client, data); err != nil {
		t.Fatalf("read INFO payload: %v", err)
	}
	if infoType := binary.BigEndian.Uint16(data[0:2]); infoType != 0 {
		t.Fatalf("info type = %d, want NBD_INFO_EXPORT (0)", infoType)
	}
	if size := binary.BigEndian.Uint64(data[2:10]); size != 1<<30 {
		t.Fatalf("advertised size = %d, want %d", size, 1<<30)
	}
	flags := TransmissionFlags(binary.BigEndian.Uint16(data[10:12]))
	if flags&FlagSendFlush == 0 {
		t.Error("NBD_FLAG_SEND_FLUSH not advertised: QEMU turns every guest fsync into a silent success without it")
	}
	if flags&FlagHASFlags == 0 {
		t.Error("NBD_FLAG_HAS_FLAGS not advertised")
	}
	if flags&(1<<8) != 0 {
		t.Error("NBD_FLAG_CAN_MULTI_CONN advertised: two connections on one volume is a corruption path")
	}

	exp := <-got
	if exp == nil || exp.Name != "/export/boot" {
		t.Fatalf("resolved export = %+v, want /export/boot", exp)
	}
}

func TestHandshake_UnknownExportIsRepErrUnknown(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	sendOption(t, client, OptGO, goPayload("/export/nope"))

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepErrUnknown {
		t.Fatalf("reply type = %#x, want NBD_REP_ERR_UNKNOWN (%#x)", typ, uint32(RepErrUnknown))
	}
}

func TestHandshake_StartTLSIsUnsupported(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	sendOption(t, client, OptStartTLS, nil)

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepErrUnsup {
		t.Fatalf("reply type = %#x, want NBD_REP_ERR_UNSUP", typ)
	}
	// The connection must not be left half-negotiated: a second option must
	// still be answerable.
	sendOption(t, client, OptAbort, nil)
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read ABORT reply after STARTTLS refusal: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepAck {
		t.Fatalf("ABORT after STARTTLS refusal: type = %#x, want ACK", typ)
	}
}

// Review Focus 1: a name length that claims more bytes than the option carried.
// The naive read is make([]byte, nameLen) followed by a slice of the payload.
func TestHandshake_GoNameLengthExceedsOptionData(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)

	// Claim a 4096-byte name in an option that carries 8 bytes.
	payload := make([]byte, 8)
	binary.BigEndian.PutUint32(payload[0:4], 4096)
	copy(payload[4:], "/export/")
	sendOption(t, client, OptGO, payload)

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepErrInvalid {
		t.Fatalf("reply type = %#x, want NBD_REP_ERR_INVALID (%#x)", typ, uint32(RepErrInvalid))
	}
}

// Review Focus 1, second shape: the option header's own data length is absurd.
// The server must not allocate what the client asked for.
func TestHandshake_HugeOptionLengthRejected(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)

	h := &OptionHeader{Magic: ihaveoptMagic, Option: OptGO, Length: 0xffffffff}
	if err := h.Encode(client); err != nil {
		t.Fatalf("encode: %v", err)
	}

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepErrTooBig {
		t.Fatalf("reply type = %#x, want NBD_REP_ERR_TOO_BIG (%#x)", typ, uint32(RepErrTooBig))
	}
}

func TestHandshake_ExportNameLegacyPath(t *testing.T) {
	server, client := pipe(t)

	got := make(chan *Export, 1)
	go func() {
		exp, err := ServeHandshake(context.Background(), server, newFakeExporter())
		if err != nil {
			got <- nil
			return
		}
		got <- exp
	}()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	// Legacy path: the name is the entire option payload and there is no
	// option reply — the export size and flags come back in lieu of one.
	sendOption(t, client, OptExportName, []byte("/export/boot"))

	var buf [10]byte
	if _, err := io.ReadFull(client, buf[:]); err != nil {
		t.Fatalf("read export reply: %v", err)
	}
	if size := binary.BigEndian.Uint64(buf[0:8]); size != 1<<30 {
		t.Fatalf("export size = %d, want %d", size, 1<<30)
	}
	if flags := TransmissionFlags(binary.BigEndian.Uint16(buf[8:10])); flags&FlagSendFlush == 0 {
		t.Error("SEND_FLUSH not advertised on the legacy path")
	}
	if exp := <-got; exp == nil || exp.Name != "/export/boot" {
		t.Fatalf("resolved export = %+v", exp)
	}
}

func TestHandshake_ListEnumeratesVolumes(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	sendOption(t, client, OptList, nil)

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read LIST reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepServer {
		t.Fatalf("LIST reply type = %#x, want NBD_REP_SERVER", typ)
	}
	length := binary.BigEndian.Uint32(hdr[16:20])
	data := make([]byte, length)
	if _, err := io.ReadFull(client, data); err != nil {
		t.Fatalf("read LIST entry: %v", err)
	}
	nameLen := binary.BigEndian.Uint32(data[0:4])
	if got := string(data[4 : 4+nameLen]); got != "/export/boot" {
		t.Fatalf("listed name = %q, want %q", got, "/export/boot")
	}
}

func TestHandshake_AbortIsAckedAndConnectionClosed(t *testing.T) {
	server, client := pipe(t)

	done := make(chan error, 1)
	go func() {
		_, err := ServeHandshake(context.Background(), server, newFakeExporter())
		done <- err
	}()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	sendOption(t, client, OptAbort, nil)

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read ABORT reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepAck {
		t.Fatalf("ABORT reply type = %#x, want ACK", typ)
	}
	select {
	case err := <-done:
		if err == nil {
			t.Fatal("ABORT returned nil; the caller cannot tell the session ended deliberately")
		}
	case <-time.After(2 * time.Second):
		t.Fatal("ServeHandshake did not return after ABORT")
	}
}

func TestHandshake_UnsupportedOptionIsRefused(t *testing.T) {
	server, client := pipe(t)

	go func() { _, _ = ServeHandshake(context.Background(), server, newFakeExporter()) }()

	readGreeting(t, client)
	sendClientFlags(t, client, 1)
	sendOption(t, client, OptStructuredReply, nil)

	var hdr [20]byte
	if _, err := io.ReadFull(client, hdr[:]); err != nil {
		t.Fatalf("read option reply: %v", err)
	}
	if typ := OptionReplyType(binary.BigEndian.Uint32(hdr[12:16])); typ != RepErrUnsup {
		t.Fatalf("NBD_OPT_STRUCTURED_REPLY reply = %#x, want NBD_REP_ERR_UNSUP", typ)
	}
}

func TestExport_TransmissionFlagsIncludeReadOnlyOnlyWhenSet(t *testing.T) {
	rw := &Export{Size: 1 << 20}
	if rw.TransmissionFlags()&FlagReadOnly != 0 {
		t.Error("writable export advertised NBD_FLAG_READ_ONLY")
	}
	ro := &Export{Size: 1 << 20, ReadOnly: true}
	if ro.TransmissionFlags()&FlagReadOnly == 0 {
		t.Error("read-only export did not advertise NBD_FLAG_READ_ONLY")
	}
	// The omissions are the load-bearing part.
	for _, f := range []TransmissionFlags{1 << 8, 1 << 7, 1 << 9, 1 << 10, 1 << 11, 1 << 12, 1 << 4} {
		if ro.TransmissionFlags()&f != 0 {
			t.Errorf("export advertised flag bit %d, which the design excludes", f)
		}
	}
}

func TestRuntimeExporter_UnknownShareIsUnknownExport(t *testing.T) {
	ex := NewRuntimeExporter(nil, Identity{UID: 65534, GID: 65534})
	if _, err := ex.Resolve(context.Background(), "/nope/boot"); !errors.Is(err, ErrUnknownExport) {
		t.Fatalf("got %v, want ErrUnknownExport", err)
	}
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `go test ./internal/adapter/nbd/ -run TestHandshake -v`
Expected: FAIL — `undefined: ServeHandshake`, `undefined: NewRuntimeExporter`, `undefined: Export`.

- [ ] **Step 3: Implement `reply.go`**

```go
func writeOptionReply(w io.Writer, opt Option, typ OptionReplyType, data []byte) error
func writeSimpleReply(w io.Writer, cookie uint64, nbdErr NBDError) error
```

`writeOptionReply` writes `OptionReplyHeader{Magic: optionReplyMagic, Option: opt, Type: typ, Length: uint32(len(data))}` then `data`. `writeSimpleReply` writes `Reply{Magic: simpleReplyMagic, Error: nbdErr, Cookie: cookie}`.

`decision:` the error-reply helpers (`writeOptionError(w, opt, typ)`) take the error type as a parameter rather than each having its own function — six near-identical functions is six places to transpose a magic.

- [ ] **Step 4: Implement `handshake.go`**

`ServeHandshake` in order:

1. Write the greeting: `nbdMagic`, `ihaveoptMagic`, then a 16-bit handshake flags field with `NBD_FLAG_FIXED_NEWSTYLE` (bit 0) and `NBD_FLAG_NO_ZEROES` (bit 1) set. Writing the client-flags read *after* the greeting is not optional — the spec has the client reply with 32-bit client flags before any option.
2. Read 32-bit client flags. Reject `NBD_FLAG_C_NO_ZEROES` when we did not offer it (we always do, so this is unreachable — assert it rather than silently accepting).
3. Loop: read `OptionHeader`. On `Magic != ihaveoptMagic` → return `ErrBadMagic`. Switch on `Option`.
4. `OptGO` / `OptInfo`: validate the payload before touching it — `Length < 6` → `RepErrInvalid`; name length `> Length-6` → `RepErrInvalid`; `Length > maxOptionData` (define `const maxOptionData = 64 << 10`) → `RepErrTooBig`. **Do the length check before allocating.** Then resolve; unknown → `RepErrUnknown`. Send `NBD_INFO_EXPORT` (type 0, 12 bytes: `uint16(0)`, `uint64(size)`, `uint16(flags)`) and, when requested, `NBD_INFO_BLOCK_SIZE` (type 3, 14 bytes: `uint16(3)`, min `uint32(1)`, preferred `uint32(4096)`, max `uint32(32<<20)`). Then `RepAck`. For `OptGO` return the export; for `OptInfo` continue the loop.

   **Both block-size values are the spec's defaults, not tuning knobs.** Minimum 1 because the engine is backed by a journal and a file, not a block device — the spec says a minimum of 1 is allowed *"for an export backed by a regular file"* and that `2^9`/`2^12` are *"more typical for an export backed by a block device"*. Advertising 4096 would make the server refuse every unaligned request, and a guest filesystem's 512-byte metadata writes are unaligned by that standard. Preferred 4096 and max `2^25` are the spec's stated defaults. **Consequence:** because we advertise the default minimum, the alignment check in the handlers is a no-op and must not be written as a 4096-multiple test.
5. `OptExportName`: the payload *is* the name. Reply with `uint64(size)`, `uint16(flags)`, then 124 zero bytes unless the client negotiated `NBD_FLAG_C_NO_ZEROES`. No option reply header on this path. Unknown export → close the connection (the spec: "it MUST terminate the session"), which is why the test asserts a non-nil error from ABORT rather than from this path.
6. `OptList`: for each name send `RepServer` with `uint32(len(name))` + name; then `RepAck`.
7. `OptAbort`: `RepAck`, then return a non-nil error (`errSessionAborted`) so the caller closes the connection.
8. `OptStartTLS`, `OptStructuredReply`, `OptListMetaContext`, `OptSetMetaContext`: `RepErrUnsup`, continue the loop.
9. Anything else: `RepErrUnsup`.

**`NBD_INFO_EXPORT` must precede the ACK.** The spec: *"The server MUST send an NBD_INFO_EXPORT information type at some point before sending an NBD_REP_ACK."* And no zero padding after the ACK.

- [ ] **Step 5: Implement `volume.go`**

`RuntimeExporter.Resolve` parses `<share>/<volume>` from the export name, resolves the share's root handle via `rt.GetRootHandle(share)` (`pkg/controlplane/runtime/runtime.go:748`), then `Lookup` down `volumes` then `<volume>` using `rt.GetMetadataService()` (`:1102`) → `metadata.Service.Lookup(ctx, dirHandle, name)` (`pkg/metadata/file_modify.go:24`). It reads `File.Size` for the export size and `File.PayloadID` for the payload ID, and `rt.GetShare(share)` (`:729`) for `ReadOnly`.

**`Lookup` needs an `*AuthContext`, and the handshake has no client identity.** Build one here with the same synthetic unprivileged identity the wire layer uses, in the local-variable style `internal/adapter/smb/handlers/auth_helper.go:50` uses (no generic helper needed in production code — `uid, gid := id.UID, id.GID` then `&uid`, `&gid`):

```go
uid, gid := r.id.UID, r.id.GID
authCtx := &metadata.AuthContext{
	Context:  ctx,
	Identity: &metadata.Identity{UID: &uid, GID: &gid},
}
```

**Do not** set `ShareReadOnly` here. This is an internal resolution against a share the operator already configured, not a client operation; the per-user ceiling that `ShareReadOnly` applies (`pkg/metadata/auth_identity.go:15`) is enforced later, on every command, from `exp.ReadOnly` via `Handler.authContext`. Setting it in both places would make the handshake refuse an export whose data the client is entitled to read. So `RuntimeExporter` needs the configured `Identity` too: `NewRuntimeExporter(rt *runtime.Runtime, id Identity) *RuntimeExporter`. That is the reason the field exists rather than the exporter reading it off the handler.

**`Handle` is what makes the write path possible, and it must be the volume FILE's handle** — not the share root, not the `volumes` directory. `PrepareWrite`/`CommitWrite` take the file handle and derive the payload ID from it; the share root's handle would resolve to a different payload. Resolve it once here and hold it for the connection's life; that is the whole point of `Export.Handle`.

A missing share, a missing `volumes` directory, or a missing file all map to `ErrUnknownExport` — the export name is client-supplied and the server must not distinguish "no such share" from "no such volume" on the wire.

`decision:` resolution happens at handshake only, never on the hot path. The size is fixed at creation, so a re-resolve would only re-read a value that cannot change; holding a file handle across a restart is exactly what §2.2 of the spec avoids.

- [ ] **Step 6: Implement `volume_test.go`'s runtime case**

`TestRuntimeExporter_UnknownShareIsUnknownExport` needs no runtime. Add one case that does, using the existing fixture:

```go
func TestRuntimeExporter_ResolvesVolumeFile(t *testing.T) {
	rt, bsID := testruntime.NewWithShareStore(t)
	store := badgertest.NewInMemory(t)
	if err := rt.RegisterMetadataStore("nbd-meta", store); err != nil {
		t.Fatal(err)
	}
	const share = "/export"
	if err := rt.AddShare(context.Background(), &runtime.ShareConfig{
		Name: share, MetadataStore: "nbd-meta", BlockStoreID: bsID,
		DefaultPermission: string(models.PermissionReadWrite),
		RootAttr:          &metadata.FileAttr{UID: 1000, GID: 1000},
	}); err != nil {
		t.Fatal(err)
	}
	root, err := rt.GetRootHandle(share)
	if err != nil {
		t.Fatal(err)
	}
	metaSvc := rt.GetMetadataService()
	authCtx := &metadata.AuthContext{
		Context:  context.Background(),
		Identity: &metadata.Identity{UID: ptr(uint32(1000)), GID: ptr(uint32(1000))},
	}
	vols, _, err := metaSvc.CreateDirectory(authCtx, root, "volumes", &metadata.FileAttr{Mode: 0o755})
	if err != nil {
		t.Fatal(err)
	}
	volsHandle, err := metadata.EncodeFileHandle(vols)
	if err != nil {
		t.Fatal(err)
	}
	created, _, err := metaSvc.CreateFile(authCtx, volsHandle, "boot", &metadata.FileAttr{Mode: 0o644})
	if err != nil {
		t.Fatal(err)
	}
	createdHandle, err := metadata.EncodeFileHandle(created)
	if err != nil {
		t.Fatal(err)
	}
	// Creation zeroes the size (ApplyCreateDefaults), so the export size is set
	// explicitly afterwards — this is the "size is fixed at creation" step.
	if _, err := metaSvc.SetFileAttributes(authCtx, createdHandle, &metadata.SetAttrs{Size: ptr(uint64(1 << 30))}); err != nil {
		t.Fatal(err)
	}

	ex := NewRuntimeExporter(rt, Identity{UID: 65534, GID: 65534})
	got, err := ex.Resolve(context.Background(), "/export/boot")
	if err != nil {
		t.Fatalf("Resolve: %v", err)
	}
	if got.Size != 1<<30 {
		t.Fatalf("size = %d, want %d", got.Size, 1<<30)
	}
	if got.PayloadID == "" {
		t.Fatal("resolved export has an empty payload ID")
	}
	if got.ReadOnly {
		t.Fatal("share is read-write but the export reports read-only")
	}
	if got.Handle == nil {
		t.Fatal("resolved export has no file handle; the write path cannot call PrepareWrite without one")
	}
}
```

Add one test helper in this file: `func ptr[T any](v T) *T { return &v }`.

Convert a `*metadata.File` to a handle with **`metadata.EncodeFileHandle(file)`** (`pkg/metadata/types.go:53`) — the same call `internal/adapter/nfs/v4/handlers/create.go:344` and `lookup.go:105` make, and the reason those adapters never need a handle-minting helper.

**Do not use `store.GenerateHandle` for this.** On Badger it ignores its `path` argument entirely: `pkg/metadata/store/badger/shares.go:21` → `basestore.GenerateHandle(ctx, shareName)` (`pkg/metadata/store/basestore/handle.go:15`) → `metadata.GenerateNewHandle` (`pkg/metadata/types.go:100`), which mints a fresh random UUID. The handle it returns would not name the file you just created, so `SetFileAttributes` would fail with an invalid-handle error and the test would be measuring the wrong thing. `EncodeFileHandle` encodes the file's own ID, which is what `GetFile` resolves against.

Do not hand-build a handle either (`[]byte(share + ":" + someUUID)`): the runtime routes by decoding it, so a fabricated handle would make the test pass and production fail.

Note: `metadata.Service.SetFileAttributes` is at `pkg/metadata/file_modify.go:339` and its size branch requires a regular file and write permission (`:720`, `:740`).

- [ ] **Step 7: Run the tests and confirm they pass**

Run: `go test ./internal/adapter/nbd/ -v`
Expected: PASS, all of Task 1 and Task 2.

- [ ] **Step 8: Verify the malformed-length tests actually bite**

In `handshake.go`, temporarily replace the `Length < 6` / name-length checks with a direct `make([]byte, nameLen)`. Run `go test ./internal/adapter/nbd/ -run TestHandshake_GoNameLengthExceedsOptionData -v` and confirm it fails — panic or `RepErrInvalid` mismatch. Restore.

- [ ] **Step 9: Commit**

```bash
git add internal/adapter/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): handshake, option haggling, and export resolution"
```

---

