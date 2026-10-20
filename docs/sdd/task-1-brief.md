## Task 1: Wire codec

**Files:**
- Create: `internal/adapter/nbd/protocol.go`
- Test: `internal/adapter/nbd/protocol_test.go`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `type Command uint16`, `type CommandFlags uint16`, `type TransmissionFlags uint16`, `type Option uint32`, `type OptionReplyType uint32`, `type NBDError uint32`
  - Constants: `CmdRead Command = 0`, `CmdWrite = 1`, `CmdDisc = 2`, `CmdFlush = 3`, `CmdTrim = 4`, `CmdCache = 5`, `CmdWriteZeroes = 6`, `CmdBlockStatus = 7`
  - `FlagFUA CommandFlags = 1 << 0`, `FlagNoHole = 1 << 1`, `FlagDF = 1 << 2`, `FlagReqOne = 1 << 3`, `FlagFastZero = 1 << 4`
  - `FlagHASFlags TransmissionFlags = 1 << 0`, `FlagReadOnly = 1 << 1`, `FlagSendFlush = 1 << 2`, `FlagSendFUA = 1 << 3`, `FlagSendTrim = 1 << 5`, `FlagSendWriteZeroes = 1 << 6`
  - `ErrEPERM NBDError = 1`, `ErrEIO = 5`, `ErrEINVAL = 22`, `ErrENOSPC = 28`, `ErrESHUTDOWN = 108`. **`NBD_EOVERFLOW` (75) and `NBD_ENOTSUP` (95) are deliberately absent**: the spec reserves both (`"The server SHOULD NOT return NBD_EOVERFLOW except as documented in response to NBD_CMD_READ when NBD_CMD_FLAG_DF is supported, or when a command without payload requests a length larger than an advertised maximum payload length"`, and the same shape for `ENOTSUP` with `FAST_ZERO`), and we advertise neither `SEND_DF` nor `SEND_FAST_ZERO`. Define them as constants only if a test needs to assert their absence; the handlers never return them.
  - `OptExportName Option = 1`, `OptAbort = 2`, `OptList = 3`, `OptStartTLS = 5`, `OptInfo = 6`, `OptGO = 7`, `OptStructuredReply = 8`, `OptListMetaContext = 9`, `OptSetMetaContext = 10`
  - `RepAck OptionReplyType = 1`, `RepServer = 2`, `RepInfo = 3`, `RepMetaContext = 4`; `RepErrUnsup = 0x80000001`, `RepErrPolicy = 0x80000002`, `RepErrInvalid = 0x80000003`, `RepErrPlatform = 0x80000004`, `RepErrTLSReqd = 0x80000005`, `RepErrUnknown = 0x80000006`, `RepErrShutdown = 0x80000007`, `RepErrBlockSizeReqd = 0x80000008`, `RepErrTooBig = 0x80000009`, `RepErrExtHeaderReqd = 0x8000000A`
  - `type Request struct { Magic uint32; Flags CommandFlags; Type Command; Cookie uint64; Offset uint64; Length uint32 }`
  - `func (r *Request) Encode(w io.Writer) error`
  - `func ReadRequest(r io.Reader) (*Request, error)`
  - `type Reply struct { Magic uint32; Error NBDError; Cookie uint64 }`
  - `func (r *Reply) Encode(w io.Writer) error`
  - `func ReadReply(r io.Reader) (*Reply, error)`
  - `type OptionHeader struct { Magic uint64; Option Option; Length uint32 }`
  - `func ReadOptionHeader(r io.Reader) (*OptionHeader, error)`
  - `func (o *OptionHeader) Encode(w io.Writer) error`
  - `type OptionReplyHeader struct { Magic uint64; Option Option; Type OptionReplyType; Length uint32 }`
  - `func (o *OptionReplyHeader) Encode(w io.Writer) error`
  - `func ReadOptionReplyHeader(r io.Reader) (*OptionReplyHeader, error)`
  - `var ErrBadMagic = errors.New("nbd: bad magic")` — the only sentinel the codec needs. There is deliberately no `ErrBadCommand`: an unknown command *value* is a well-formed 28-byte header that the dispatcher answers with `ErrEINVAL`, not a decode failure. Validating it at decode time would put a protocol rule in the codec and make the dispatcher's `default` branch unreachable.

- [ ] **Step 1: Write the failing round-trip test**

Create `internal/adapter/nbd/protocol_test.go`. All values below are the spec's, not invented.

```go
package nbd

import (
	"bytes"
	"errors"
	"testing"
)

func TestRequestRoundTrip(t *testing.T) {
	want := &Request{
		Magic:  requestMagic,
		Flags:  FlagFUA,
		Type:   CmdWrite,
		Cookie: 0xdeadbeefcafef00d,
		Offset: 4096,
		Length: 8192,
	}
	var buf bytes.Buffer
	if err := want.Encode(&buf); err != nil {
		t.Fatalf("Encode: %v", err)
	}
	if got := buf.Len(); got != 28 {
		t.Fatalf("encoded request is %d bytes, want 28", got)
	}
	got, err := ReadRequest(&buf)
	if err != nil {
		t.Fatalf("ReadRequest: %v", err)
	}
	if *got != *want {
		t.Fatalf("round trip: got %+v, want %+v", got, want)
	}
}

func TestRequestBadMagic(t *testing.T) {
	buf := bytes.NewReader([]byte{
		0xde, 0xad, 0xbe, 0xef, // not NBD_REQUEST_MAGIC
		0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
		0, 0, 0, 0, 0, 0, 0, 0,
	})
	if _, err := ReadRequest(buf); !errors.Is(err, ErrBadMagic) {
		t.Fatalf("got %v, want ErrBadMagic", err)
	}
}

func TestReplyRoundTrip(t *testing.T) {
	want := &Reply{Magic: simpleReplyMagic, Error: ErrEIO, Cookie: 0x1122334455667788}
	var buf bytes.Buffer
	if err := want.Encode(&buf); err != nil {
		t.Fatalf("Encode: %v", err)
	}
	if got := buf.Len(); got != 16 {
		t.Fatalf("encoded reply is %d bytes, want 16", got)
	}
	got, err := ReadReply(&buf)
	if err != nil {
		t.Fatalf("ReadReply: %v", err)
	}
	if *got != *want {
		t.Fatalf("round trip: got %+v, want %+v", got, want)
	}
}

func TestOptionHeaderRoundTrip(t *testing.T) {
	want := &OptionHeader{Magic: ihaveoptMagic, Option: OptGO, Length: 42}
	var buf bytes.Buffer
	if err := want.Encode(&buf); err != nil {
		t.Fatalf("Encode: %v", err)
	}
	if got := buf.Len(); got != 16 {
		t.Fatalf("encoded option header is %d bytes, want 16", got)
	}
	got, err := ReadOptionHeader(&buf)
	if err != nil {
		t.Fatalf("ReadOptionHeader: %v", err)
	}
	if *got != *want {
		t.Fatalf("round trip: got %+v, want %+v", got, want)
	}
}

func TestOptionReplyHeaderIs20Bytes(t *testing.T) {
	h := &OptionReplyHeader{Magic: optionReplyMagic, Option: OptGO, Type: RepAck, Length: 0}
	var buf bytes.Buffer
	if err := h.Encode(&buf); err != nil {
		t.Fatalf("Encode: %v", err)
	}
	if got := buf.Len(); got != 20 {
		t.Fatalf("encoded option reply header is %d bytes, want 20", got)
	}
}

// The magic values are the contract with every client. A transposed byte here
// is invisible until a real client refuses to negotiate.
func TestMagicValues(t *testing.T) {
	cases := []struct {
		name string
		got  uint64
		want uint64
	}{
		{"NBDMAGIC", nbdMagic, 0x4e42444d41474943},
		{"IHAVEOPT", ihaveoptMagic, 0x49484156454f5054},
		{"option reply", optionReplyMagic, 0x3e889045565a9},
	}
	for _, c := range cases {
		if c.got != c.want {
			t.Errorf("%s = %#x, want %#x", c.name, c.got, c.want)
		}
	}
	if requestMagic != 0x25609513 {
		t.Errorf("request magic = %#x, want 0x25609513", requestMagic)
	}
	if simpleReplyMagic != 0x67446698 {
		t.Errorf("simple reply magic = %#x, want 0x67446698", simpleReplyMagic)
	}
}

// A truncated read must surface as an error, never as a zero-valued header that
// the dispatcher then acts on.
func TestReadRequestTruncated(t *testing.T) {
	buf := bytes.NewReader([]byte{0x25, 0x60, 0x95, 0x13, 0x00})
	if _, err := ReadRequest(buf); err == nil {
		t.Fatal("truncated header decoded without error")
	}
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `go test ./internal/adapter/nbd/ -run 'TestRequest|TestReply|TestOption|TestMagic' -v`
Expected: FAIL — `undefined: requestMagic`, `undefined: Request`, and the rest.

- [ ] **Step 3: Implement `protocol.go`**

Write the constants and the four encode/decode pairs. Bodies are determined by the header layouts in the spec, so they are not transcribed here. Two constraints that are not visible from the layouts:

- **Decode from a `[]byte` read with `io.ReadFull`, not field-by-field `binary.Read` calls.** `io.ReadFull` on a fixed-size buffer gives one error path for a truncated header and guarantees no partial decode is acted on. Field-by-field reads produce a half-populated `Request` on error, which is the shape that makes `ErrBadMagic` unreachable in practice.
- **`ReadRequest` checks the magic before populating anything else**, returning `ErrBadMagic` wrapped with the offending value: `fmt.Errorf("%w: %#x", ErrBadMagic, magic)`.
- `ReadOptionReplyHeader` is needed by tests, not by the server. Say so in its doc comment rather than leaving a reader to wonder.

Use `encoding/binary.BigEndian` throughout — NBD is big-endian on the wire.

- [ ] **Step 4: Run the tests and confirm they pass**

Run: `go test ./internal/adapter/nbd/ -run 'TestRequest|TestReply|TestOption|TestMagic' -v`
Expected: PASS, 7 tests.

- [ ] **Step 5: Verify the magic-value test actually bites**

Temporarily change `requestMagic` to `0x25609514`, run `go test ./internal/adapter/nbd/ -run TestMagicValues -v`, and confirm it fails naming the request magic. Revert.

- [ ] **Step 6: Commit**

```bash
git add internal/adapter/nbd/protocol.go internal/adapter/nbd/protocol_test.go
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): wire codec for the NBD protocol"
```

---

