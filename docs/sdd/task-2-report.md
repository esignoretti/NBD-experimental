# Task 2 report — Handshake and export resolution

**Status:** DONE_WITH_CONCERNS (two brief-vs-code notes below; no defect in the work)
**Commit:** `c13b595fc33a19e7207a0e1e4c1d3b7a302cdd41` — `feat(nbd): handshake, option haggling, and export resolution`
**Branch:** `nbd-block-protocol` (worktree `<repo>`), base `90cb65bc`
**Signed:** yes, SSH signature present (`gpgsig` block in the commit object; `git log --format=%G?` reports `N` only because `gpg.ssh.allowedSignersFile` is not configured in this environment, not because the signature is absent). `bash test/no-ai-attribution.sh 90cb65bc` → `no AI attribution in 1 commit(s)`.

## Files

- `internal/adapter/nbd/reply.go` (new, 48 lines)
- `internal/adapter/nbd/handshake.go` (new, 356 lines)
- `internal/adapter/nbd/volume.go` (new, 198 lines)
- `internal/adapter/nbd/volume_test.go` (new, 80 lines)

`internal/adapter/nbd/handshake_test.go` was **not** touched: it was already committed at `90cb65bc` as the plan's own verbatim fence, and `diff` of the brief's step-1 Go block against the file is byte-identical. `git diff --quiet HEAD -- internal/adapter/nbd/handshake_test.go` → clean.

## Step 1 — pre-flight verification (not rewritten)

Extracted the brief's first ```` ```go ```` fence to `/tmp/brief_step1.go` and diffed it against the committed test file:

```
$ diff /tmp/brief_step1.go internal/adapter/nbd/handshake_test.go && echo IDENTICAL
IDENTICAL
```

## Step 2 — RED (actual failure text)

Command: `go test ./internal/adapter/nbd/ -run TestHandshake -v`

```
# github.com/marmos91/dittofs/internal/adapter/nbd [github.com/marmos91/dittofs/internal/adapter/nbd.test]
internal/adapter/nbd/handshake_test.go:15:22: undefined: Export
internal/adapter/nbd/handshake_test.go:18:66: undefined: Export
internal/adapter/nbd/handshake_test.go:22:14: undefined: ErrUnknownExport
internal/adapter/nbd/handshake_test.go:34:44: undefined: Export
internal/adapter/nbd/handshake_test.go:144:13: undefined: ServeHandshake
internal/adapter/nbd/handshake_test.go:164:20: undefined: Export
internal/adapter/nbd/handshake_test.go:166:15: undefined: ServeHandshake
internal/adapter/nbd/handshake_test.go:216:21: undefined: ServeHandshake
internal/adapter/nbd/handshake_test.go:234:21: undefined: ServeHandshake
internal/adapter/nbd/handshake_test.go:263:21: undefined: ServeHandshake
internal/adapter/nbd/handshake_test.go:263:21: too many errors
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd [build failed]
FAIL
```

## Step 7 — GREEN

Command: `go test ./internal/adapter/nbd/ -v`

```
--- PASS: TestHandshake_GreetingAdvertisesFixedNewstyle (0.00s)
--- PASS: TestHandshake_GoReturnsExportInfoThenAck (0.00s)
--- PASS: TestHandshake_UnknownExportIsRepErrUnknown (0.00s)
--- PASS: TestHandshake_StartTLSIsUnsupported (0.00s)
--- PASS: TestHandshake_GoNameLengthExceedsOptionData (0.00s)
--- PASS: TestHandshake_HugeOptionLengthRejected (0.00s)
--- PASS: TestHandshake_ExportNameLegacyPath (0.00s)
--- PASS: TestHandshake_ListEnumeratesVolumes (0.00s)
--- PASS: TestHandshake_AbortIsAckedAndConnectionClosed (0.00s)
--- PASS: TestHandshake_UnsupportedOptionIsRefused (0.00s)
--- PASS: TestExport_TransmissionFlagsIncludeReadOnlyOnlyWhenSet (0.00s)
--- PASS: TestRuntimeExporter_UnknownShareIsUnknownExport (0.00s)
--- PASS: TestRequestRoundTrip (0.00s)
--- PASS: TestRequestBadMagic (0.00s)
--- PASS: TestReplyRoundTrip (0.00s)
--- PASS: TestOptionHeaderRoundTrip (0.00s)
--- PASS: TestOptionReplyHeaderIs20Bytes (0.00s)
--- PASS: TestMagicValues (0.00s)
--- PASS: TestReadRequestTruncated (0.00s)
--- PASS: TestRuntimeExporter_ResolvesVolumeFile (0.14s)
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	1.471s
```

20 tests, 0 failures. Also: `-count=3` → ok; `-race -count=1` → ok.

## Step 8 — the malformed-length tests bite

Three reverts, each restored afterwards and verified byte-identical to the committed revision (`diff` → `RESTORED_IDENTICAL`).

**8a. Replace the length checks with a direct `make([]byte, nameLen)`** (the brief's step 8). This fails as a panic, which the brief accepts ("panic or `RepErrInvalid` mismatch") but which is *not* an assertion failure, so 8b and 8c below give the assertion-level proof the dispatch asked for.

Command: `go test ./internal/adapter/nbd/ -run TestHandshake_GoNameLengthExceedsOptionData -v`

```
=== RUN   TestHandshake_GoNameLengthExceedsOptionData
panic: runtime error: slice bounds out of range [4102:8]

goroutine 55 [running]:
github.com/marmos91/dittofs/internal/adapter/nbd.parseGoPayload({0x5a20121bc040, 0x8, 0x8})
	<repo>/internal/adapter/nbd/handshake.go:285 +0x204
github.com/marmos91/dittofs/internal/adapter/nbd.serveGoOption(...)
	<repo>/internal/adapter/nbd/handshake.go:248 +0x48
github.com/marmos91/dittofs/internal/adapter/nbd.ServeHandshake(...)
	<repo>/internal/adapter/nbd/handshake.go:178 +0x1f4
github.com/marmos91/dittofs/internal/adapter/nbd.TestHandshake_GoNameLengthExceedsOptionData.func1()
	<repo>/internal/adapter/nbd/handshake_test.go:263 +0x104
```

**8b. Fail ON THE ASSERTION — answer the malformed payload with the wrong error type** (swap `RepErrInvalid`→`RepErrUnsup` in `serveGoOption` and `RepErrTooBig`→`RepErrUnsup` in the length guard):

Command: `go test ./internal/adapter/nbd/ -run 'TestHandshake_GoNameLengthExceedsOptionData|TestHandshake_HugeOptionLengthRejected' -v`

```
=== RUN   TestHandshake_GoNameLengthExceedsOptionData
    handshake_test.go:279: reply type = 0x80000001, want NBD_REP_ERR_INVALID (0x80000003)
--- FAIL: TestHandshake_GoNameLengthExceedsOptionData (0.00s)
=== RUN   TestHandshake_HugeOptionLengthRejected
    handshake_test.go:303: reply type = 0x80000001, want NBD_REP_ERR_TOO_BIG (0x80000009)
--- FAIL: TestHandshake_HugeOptionLengthRejected (0.00s)
FAIL
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	1.048s
```

**8c. Drop `FlagSendFlush` from `(*Export).TransmissionFlags`** — the failure this whole task exists to prevent, and the one an assertion catches on both handshake paths:

Command: `go test ./internal/adapter/nbd/ -run 'TestHandshake_GoReturnsExportInfoThenAck|TestHandshake_ExportNameLegacyPath' -v`

```
=== RUN   TestHandshake_GoReturnsExportInfoThenAck
    handshake_test.go:198: NBD_FLAG_SEND_FLUSH not advertised: QEMU turns every guest fsync into a silent success without it
--- FAIL: TestHandshake_GoReturnsExportInfoThenAck (0.00s)
=== RUN   TestHandshake_ExportNameLegacyPath
    handshake_test.go:334: SEND_FLUSH not advertised on the legacy path
--- FAIL: TestHandshake_ExportNameLegacyPath (0.00s)
FAIL
FAIL	github.com/marmos91/dittofs/internal/adapter/nbd	1.041s
```

## Commands run

```
go build ./...                                             # OK
go vet ./internal/adapter/nbd/                             # OK
gofmt -l internal/adapter/nbd/                             # empty
go test ./internal/adapter/nbd/ -run TestHandshake -v       # RED (step 2)
go test ./internal/adapter/nbd/ -v                          # GREEN, 20/20
go test ./internal/adapter/nbd/ -count=3                    # ok
go test ./internal/adapter/nbd/ -race -count=1              # ok
GOTOOLCHAIN=go1.26.0 golangci-lint run ./internal/adapter/nbd/...
git add internal/adapter/nbd/ && git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): handshake, option haggling, and export resolution"
bash test/no-ai-attribution.sh 90cb65bc
```

Lint: production files are clean. Four `errcheck` findings remain, all in the plan-owned `handshake_test.go` (`defer ln.Close()` at :54, `client.Close()` at :75/:153, `a.conn.Close()` at :76) — that file is the plan's own verbatim fence committed at `90cb65bc` and the brief says not to rewrite it, so they are left as-is. `golangci-lint` needs `GOTOOLCHAIN=go1.26.0` here: the installed binary is built against go1.26 while the default toolchain resolves to go1.27.1, and without the override it panics inside `go/types` before reporting anything (the same environmental mismatch Task 1 recorded).

## Implementation notes

- **`Export.Handle` is the volume FILE's handle**, from `metadata.EncodeFileHandle(file)` — not the share root's and not the `volumes` directory's. `store.GenerateHandle` is not used anywhere: on Badger it mints a fresh random UUID and would not name the file.
- **`TransmissionFlags`** = `FlagHASFlags | FlagSendFlush | FlagSendFUA | FlagSendTrim | FlagSendWriteZeroes`, plus `FlagReadOnly` only when `ReadOnly`. Bit 4 and bits 7–12 are never set.
- **`NBD_INFO_EXPORT` always precedes the ACK**, whether or not the client requested it; `NBD_INFO_BLOCK_SIZE` (min 1, preferred 4096, max 32 MiB) only when requested. No zero padding after the ACK.
- **Length checks precede allocation**: `Length > maxOptionData` (64 KiB) → `RepErrTooBig`; a payload under 6 bytes, a name length past the payload, or an information-request count past the remaining bytes → `RepErrInvalid`.
- **`ShareReadOnly` is not set** on the handshake's `AuthContext`; `ReadOnly` is read from `rt.GetShare(share)` and carried on the export for the per-command ceiling.
- `writeSimpleReply` carries `//nolint:unused` with a one-line reason: its first caller is Task 4's dispatcher, and the brief declares it in this task's Interfaces. Without the directive `unused` fails the package lint.

## Concerns / judgement calls (none block acceptance)

1. **Brief is silent on what happens after `RepErrTooBig`; I close the session.** The brief says `Length > maxOptionData` → `RepErrTooBig` and "do the length check before allocating", but not whether the loop continues. Continuing would require consuming the announced bytes to keep the stream aligned — the one length the server has just refused to believe — so the reply is written and then the handshake returns a non-nil error, with a `decision:` comment recording that. The test asserts only the reply type, so it passes either way; if the intent was to keep the session alive, the change is three lines.
2. **`Exporter.List` errors end the handshake rather than being answered with an empty list.** A client reads an empty `NBD_OPT_LIST` as "this server has no exports", which is a different and wrong statement, and the brief gives no error reply for LIST. No test covers this path (the fake exporter never fails).
3. **The client's `NBD_FLAG_C_FIXED_NEWSTYLE` bit is not validated.** The brief's step 2 asks only that `NBD_FLAG_C_NO_ZEROES` be rejected when not offered, which is implemented; an unused `clientFlagFixedNewstyle` constant was removed rather than kept dead. A client that omits the bit is served anyway.
4. **`RuntimeExporter.List` is implemented but untested**, and it enumerates every share's `volumes` directory with pagination. The brief specifies `List` in Interfaces and the wire behaviour in step 6 but prescribes no test for the runtime implementation. It uses `rt.ListShares()` + `Lookup` + `ReadDirectory`, which no other case in this task exercises.
5. **`RuntimeExporter` gained an unexported `errNoRuntime`** so that `List` on a nil runtime is not reported as "no such export". It is not in the brief's Interfaces list; it is unexported, so no interface widens. `Resolve` keeps returning `ErrUnknownExport` for a nil runtime, which is what `TestRuntimeExporter_UnknownShareIsUnknownExport` requires.
6. `NewRuntimeExporter(nil, …)` is only reachable from tests; `Resolve` guards it rather than panicking because that is what the brief's own test constructs.

## Self-review

Read the full diff (`git show --stat HEAD`, 4 files, 682 insertions) before committing. Checked: no file outside the brief's Files list was modified; `handshake_test.go` is unchanged; no new dependency (imports are `context`, `encoding/binary`, `errors`, `fmt`, `io`, `net`, `slices`, plus the repo's `pkg/metadata` and `pkg/controlplane/runtime`); no `TODO`, no placeholder; no issue/PR/CI/phase id in any comment; the `decision:` marker is present at the two places the brief asks for it (`writeOptionError` in `reply.go`, resolve-once in `volume.go`) and at the two places a judgement was made (over-long option closes the session; `ShareReadOnly` deliberately unset).
