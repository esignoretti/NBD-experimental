# Task 1 report — Wire codec

**Status:** DONE
**Commit:** `0f8d7d61` — `feat(nbd): wire codec for the NBD protocol`
**Branch:** `nbd-block-protocol` (worktree `<repo>`), base `cc7b34cb`

## What I implemented

Created the NBD wire codec for `internal/adapter/nbd`, package `nbd`:

- **Magics** (`uint64` handshake magics, `uint32` transmission magics):
  `nbdMagic` = `0x4e42444d41474943`, `ihaveoptMagic` = `0x49484156454f5054`,
  `optionReplyMagic` = `0x3e889045565a9`, `requestMagic` = `0x25609513`,
  `simpleReplyMagic` = `0x67446698`.
- **`type Command uint16`** — `CmdRead` 0 … `CmdBlockStatus` 7.
- **`type CommandFlags uint16`** — `FlagFUA` `1<<0` … `FlagFastZero` `1<<4`.
- **`type TransmissionFlags uint16`** — `FlagHASFlags` `1<<0`, `FlagReadOnly` `1<<1`,
  `FlagSendFlush` `1<<2`, `FlagSendFUA` `1<<3`, `FlagSendTrim` `1<<5`,
  `FlagSendWriteZeroes` `1<<6`. Bit 4 (rotational) and bits 7+ deliberately absent.
- **`type NBDError uint32`** — `ErrEPERM` 1, `ErrEIO` 5, `ErrEINVAL` 22, `ErrENOSPC` 28,
  `ErrESHUTDOWN` 108. `NBD_EOVERFLOW` (75) and `NBD_ENOTSUP` (95) deliberately absent
  (documented inline: the adapter advertises neither `SEND_DF` nor `SEND_FAST_ZERO`).
- **`type Option uint32`** and **`type OptionReplyType uint32`** with the full option /
  reply constant sets from the brief (`OptExportName` … `OptSetMetaContext`,
  `RepAck` … `RepErrExtHeaderReqd`).
- **`ErrBadMagic`** — the only sentinel. No `ErrBadCommand` (documented with a
  `decision:` marker: an unknown command value is a well-formed 28-byte header that the
  dispatcher answers with `EINVAL`; rejecting it at decode would leave the dispatcher's
  `default` branch unreachable).
- **Header types + codecs**: `Request`/`ReadRequest`/`(*Request).Encode`,
  `Reply`/`ReadReply`/`(*Reply).Encode`, `OptionHeader`/`ReadOptionHeader`/`Encode`,
  `OptionReplyHeader`/`ReadOptionReplyHeader`/`Encode`. All use `encoding/binary.BigEndian`.

Design constraints honoured from the brief:

- Decode reads the whole fixed-size header with `io.ReadFull` into a `[N]byte`, then
  populates fields — never field-by-field `binary.Read`. One error path for truncation,
  no half-populated struct on error.
- `ReadRequest` checks magic before populating anything and returns
  `fmt.Errorf("%w: %#x", ErrBadMagic, magic)`.
- `ReadOptionReplyHeader` carries a doc comment stating it exists for tests, not the
  server.

## Files changed

- `internal/adapter/nbd/protocol.go` (new, 284 lines)
- `internal/adapter/nbd/protocol_test.go` (new, 124 lines)

The test file is byte-for-byte identical to the brief's verbatim test block
(verified with `diff` against the fenced block in the plan: `TEST FILE MATCHES BRIEF VERBATIM`).

## TDD evidence

### RED — step 2, before any implementation

Command:
```
go test ./internal/adapter/nbd/ -run 'TestRequest|TestReply|TestOption|TestMagic' -v
```

Observed output:
```
# github.com/marmos91/dittofs/internal/adapter/nbd [github.com/marmos91/dittofs/internal/adapter/nbd.test]
internal/adapter/nbd/protocol_test.go:10:11: undefined: Request
internal/adapter/nbd/protocol_test.go:11:11: undefined: requestMagic
internal/adapter/nbd/protocol_test.go:12:11: undefined: FlagFUA
internal/adapter/nbd/protocol_test.go:13:11: undefined: CmdWrite
internal/adapter/nbd/protocol_test.go:25:14: undefined: ReadRequest
internal/adapter/nbd/protocol_test.go:40:15: undefined: ReadRequest
internal/adapter/nbd/protocol_test.go:40:49: undefined: ErrBadMagic
internal/adapter/nbd/protocol_test.go:46:11: undefined: Reply
internal/adapter/nbd/protocol_test.go:46:24: undefined: simpleReplyMagic
internal/adapter/nbd/protocol_test.go:46:49: undefined: ErrEIO
internal/adapter/nbd/protocol_test.go:46:49: too many errors
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd [build failed]
FAIL
```

Why expected: the test was written first and the package had no `protocol.go`; every
exported name it references (`Request`, `ReadRequest`, `ErrBadMagic`, …) was undefined.
This is the brief's step-2 expectation (`undefined: requestMagic`, `undefined: Request`,
and the rest) exactly.

### RED — step 5, magic-value test bites

This is the proof the brief requires: a transposition must fail **on the assertion**,
not as a compile error. I temporarily changed the constant in `protocol.go` line 22:

```
requestMagic uint32 = 0x25609514   // was 0x25609513
```

Command:
```
go test ./internal/adapter/nbd/ -run TestMagicValues -v
```

Observed output:
```
=== RUN   TestMagicValues
    protocol_test.go:110: request magic = 0x25609514, want 0x25609513
--- FAIL: TestMagicValues (0.00s)
FAIL
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	0.391s
FAIL
```

Why an assertion failure rather than a compile error is the proof required: the test
still compiles (the identifier resolves), so the run reaches `TestMagicValues` and fails
at its own check. A compile error would only prove the symbol was renamed or removed —
it would say nothing about whether the asserted *value* is checked. `--- FAIL:
TestMagicValues` with the message naming the request magic proves the assertion is live
and that a one-bit transposition in the wire contract is caught.

Reverted the constant and re-ran: `--- PASS: TestMagicValues`.

## GREEN — after implementation and after restore

Command:
```
go test ./internal/adapter/nbd/ -v -count=1
```

Observed output:
```
=== RUN   TestRequestRoundTrip
--- PASS: TestRequestRoundTrip (0.00s)
=== RUN   TestRequestBadMagic
--- PASS: TestRequestBadMagic (0.00s)
=== RUN   TestReplyRoundTrip
--- PASS: TestReplyRoundTrip (0.00s)
=== RUN   TestOptionHeaderRoundTrip
--- PASS: TestOptionHeaderRoundTrip (0.00s)
=== RUN   TestOptionReplyHeaderIs20Bytes
--- PASS: TestOptionReplyHeaderIs20Bytes (0.00s)
=== RUN   TestMagicValues
--- PASS: TestMagicValues (0.00s)
=== RUN   TestReadRequestTruncated
--- PASS: TestReadRequestTruncated (0.00s)
PASS
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	0.676s
```

All 7 tests pass:

| Test | Result |
|---|---|
| `TestRequestRoundTrip` | PASS |
| `TestRequestBadMagic` | PASS |
| `TestReplyRoundTrip` | PASS |
| `TestOptionHeaderRoundTrip` | PASS |
| `TestOptionReplyHeaderIs20Bytes` | PASS |
| `TestMagicValues` | PASS |
| `TestReadRequestTruncated` | PASS |

## Verification commands

| Command | Result |
|---|---|
| `go build ./...` | BUILD OK |
| `go vet ./internal/adapter/nbd/` | VET OK |
| `go test ./internal/adapter/nbd/ -v -count=1` | PASS, 7/7 |
| `gofmt -l internal/adapter/nbd/` | no output (clean) |
| `golangci-lint run ./internal/adapter/nbd/...` | `0 issues.` (exit 0) |
| `bash test/no-ai-attribution.sh cc7b34cb` | `no AI attribution in 1 commit(s)` (exit 0) |

Note on lint: the repo-wide `make lint` panics in this environment with
`panic: file requires newer Go version go1.27 (application built with go1.26)` — an
environmental golangci-lint/Go toolchain mismatch (the same panic occurs on untouched
packages such as `./internal/adapter/nfs/...`). Package-scoped
`golangci-lint run ./internal/adapter/nbd/...` completes normally with 0 issues, which is
the meaningful result for this task.

Signing: commit made with
`git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S`.
`git cat-file commit HEAD` shows an SSH `gpgsig` header, same as the two prior branch
commits (`cc7b34cb`, `8456999e`). Local `git log --show-signature` reports "No signature"
only because `gpg.ssh.allowedSignersFile` is unset in this checkout — a verification-config
gap affecting all three commits equally, not a missing signature.

## Self-review findings

- Read the full diff. Struct field order, widths, and offsets match the brief's
  declarations and the spec's header layouts (request 28 B, reply 16 B, option header
  16 B, option reply header 20 B) — each asserted by a test.
- Confirmed every name the brief lists under "Produces" is present (scripted check over
  the full constant/type/function list: no missing entries).
- Confirmed no forbidden comment references: no issue/PR numbers, CI names, phase IDs,
  or plan/task references in either file.
- Confirmed `ErrBadCommand` is absent and `ErrEOVERFLOW`/`ErrENOTSUP` are absent
  (the only mentions of 75/95 are the explanatory comment).
- Ran `gofmt -w` once after the first write (a const block was misaligned); the committed
  file is gofmt-clean.

## Concerns

1. **Scope of option-header types.** The dispatch message glossed `OptionHeader` /
   `OptionReplyHeader` as "Task 2's, not yours", but the brief's own "Produces" list,
   the plan's Task 1 "Produces" list, and the plan's Task 2 "Consumes: Task 1's
   constants, `OptionHeader`, `OptionReplyHeader`, `TransmissionFlags`" all assign them
   to Task 1. The brief is authoritative and the verbatim test
   (`TestOptionHeaderRoundTrip`, `TestOptionReplyHeaderIs20Bytes`) references them
   directly, so I implemented them. Flagging in case the controller wants to reconcile
   the message wording.
2. **Environment lint mismatch** described above: repo-wide `make lint` cannot run here.
   Not caused by this change.
3. No `ErrBadCommand` sentinel and no command-value validation at decode time, by design;
   a later dispatcher task must own the `default` → `EINVAL` branch.

## Not done (by design)

- No `NBD_EOVERFLOW` / `NBD_ENOTSUP` constants.
- No `ErrBadCommand` sentinel.
- No `BLOCK_STATUS` / structured-reply surface (residency-truth gate, later phase).
- No option-payload parsing (Task 2's handshake).
