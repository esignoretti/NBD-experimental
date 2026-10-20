# Task 2 fix report — disabled-share gate and the Handle contract

**Status:** DONE
**Commits:**
- `d9f8e7afb6f4c340a5b545ced6aee3a5dbbb64f9` — `fix(nbd): refuse and omit a disabled share at export resolution` (`internal/adapter/nbd/volume.go`, `internal/adapter/nbd/volume_test.go`; 2 files, +97)
- `df69c170` — `docs(nbd): plan the disabled-share gate and the file-handle assertion` (`docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md`; 1 file, +78 −1)

Both signed (SSH signature present in the commit objects). `bash test/no-ai-attribution.sh c13b595f` → `no AI attribution in 3 commit(s)`. Working tree clean, nothing pushed.

## Finding 1 — a disabled share is still exportable

### Gate added in `Resolve` (`internal/adapter/nbd/volume.go`, immediately after `GetShare`)

```go
	sh, err := r.rt.GetShare(share)
	if err != nil {
		return nil, ErrUnknownExport
	}
	// A disabled share is being quiesced, and no new client may enter it.
	//
	// decision: the gate belongs here because the handshake is the only point
	// at which NBD admits a client; NFS MOUNT, SMB TREE_CONNECT and the NFSv4
	// share-entry check each refuse a disabled share when access is
	// established, and the write path that follows does not re-consult the
	// flag. The refusal is reported as an unknown export rather than as an
	// error of its own, because the export name is client-supplied and the
	// server must not tell a client whether a share exists but is quiesced.
	if !sh.Enabled {
		return nil, ErrUnknownExport
	}
```

### Skip added in `List` (top of the `r.rt.ListShares()` loop)

```go
		sh, err := r.rt.GetShare(share)
		if err != nil {
			return nil, err
		}
		// A disabled share is skipped for the same reason Resolve refuses it:
		// advertising an export the server would then refuse to resolve is a
		// worse answer than not advertising it.
		if !sh.Enabled {
			continue
		}
```

## Finding 2 — the Handle contract was asserted only as non-nil

`internal/adapter/nbd/volume_test.go` now compares against the handle the fixture already computed at `createdHandle`:

```go
	if got.Handle == nil {
		t.Fatal("resolved export has no file handle; the write path cannot call PrepareWrite without one")
	}
	// The handle must be the volume file's own, not the share root's and not
	// the volumes directory's: PrepareWrite/CommitWrite derive the payload ID
	// from it, so either of the other two would address different bytes.
	if !bytes.Equal(got.Handle, createdHandle) {
		t.Fatalf("export handle = %q, want the volume file's handle %q", got.Handle, createdHandle)
	}
```

The nil check is kept ahead of the equality check so a nil handle still reports the informative "no file handle" message rather than an empty-vs-real comparison. `"bytes"` and `"errors"` were added to the imports.

## Tests added

- `TestRuntimeExporter_DisabledShareIsRefusedAndUnlisted` (`internal/adapter/nbd/volume_test.go`) — same fixture as `TestRuntimeExporter_ResolvesVolumeFile`. Resolves and lists the export while the share is enabled, asserts the enabled `List` returns exactly `[/export/boot]`, then calls `rt.SetEnabledForTesting(share, false)` and asserts `Resolve` returns an error matching `ErrUnknownExport` (`errors.Is`) **and** `List` returns no names. Both halves of the gate are pinned in one test, as allowed.
- `TestRuntimeExporter_ResolvesVolumeFile` strengthened in place with the `bytes.Equal` assertion above.

### Fixture correction that the gate forced

`AddShare` copies `Enabled` verbatim from `ShareConfig` (`pkg/controlplane/runtime/shares/lifecycle.go:548`) and nothing defaults it, so the existing fixture — which set only `Name`, `MetadataStore`, `BlockStoreID`, `DefaultPermission`, `RootAttr` — built a **disabled** share. Adding the gate made `TestRuntimeExporter_ResolvesVolumeFile` fail with `ErrUnknownExport` until the fixture set `Enabled: true`, which is what the plan's own other fixtures do. Both test functions now pass `Enabled: true` with a comment stating why. This is worth flagging to the whole-branch review: any other test that builds a share through `runtime.ShareConfig` without `Enabled: true` is building a disabled one, and Task 3's adapter tests and Task 4's connection tests will need it too.

## RED / GREEN for the new test

**GREEN** — `go test ./internal/adapter/nbd/ -v -count=1`: 21 tests, 0 failures. `go test ./internal/adapter/nbd/... -race -count=1` → ok (3.466s). `go build ./...` → ok. `gofmt -l internal/adapter/nbd/` → empty. `GOTOOLCHAIN=go1.26.0 golangci-lint run ./internal/adapter/nbd/...` → **0 issues** (the four `errcheck` hits that were in `handshake_test.go` were fixed separately by the controller in `283105a1`).

**RED, gate removed — `Resolve` half.** Deleted the `if !sh.Enabled { return nil, ErrUnknownExport }` block, rebuilt, ran `go test ./internal/adapter/nbd/ -run TestRuntimeExporter_DisabledShareIsRefusedAndUnlisted -v`:

```
    volume_test.go:148: Resolve on a disabled share: got <nil>, want ErrUnknownExport
```

Fails **on the assertion**, no compile error, no unused-import error.

**RED, gate removed — `List` half.** Replaced the `if !sh.Enabled { continue }` block with `_ = sh` (keeping the `GetShare` call, so the `err` variable stays used and the build stays clean), ran the same test:

```
    volume_test.go:153: List on a disabled share = [/export/boot], want no exports
```

Again an assertion failure. Both halves were restored afterwards and verified byte-identical (`diff` → `RESTORED_IDENTICAL`; `git status` clean).

**RED, Handle assertion.** Forced the wrong handle by appending `handle = root` after the `EncodeFileHandle(file)` call, ran `-run TestRuntimeExporter_ResolvesVolumeFile -v`:

```
    volume_test.go:90: export handle = "/export:01a638f5-fac5-414d-9ff2-24715700247c", want the volume file's handle "/export:6b7c37da-ac7f-4f7e-bc87-0b15ec4e7cdf"
--- FAIL: TestRuntimeExporter_ResolvesVolumeFile (0.10s)
```

The share root's handle is a real, non-nil handle that the old non-nil check accepted — this is exactly the case Finding 2 described. Restored and verified identical.

## Plan edits (`docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md`)

Two edits only, both inside Task 2's own steps; nothing else in the plan was touched (`git diff -U0` shows hunks at lines 856, 874, 879, 933 and 1005, all within Steps 5 and 6).

1. **Step 5** — appended two paragraphs after the "A missing share, a missing `volumes` directory, or a missing file all map to `ErrUnknownExport`" sentence: one stating that `Resolve` refuses a disabled share and `List` skips one, naming the three precedents by path and line (`internal/adapter/nfs/mount/handlers/mount.go:114` `MountErrAccess`, `internal/adapter/smb/handlers/tree_connect.go:119` `STATUS_NETWORK_NAME_DELETED`, `internal/adapter/nfs/v4/handlers/helpers.go:226` `NFS4ERR_STALE`), quoting `pkg/controlplane/runtime/shares/service.go:26-29`, and giving the reason the refusal is reported as an unknown export; and one stating that the fixture must set `Enabled: true` because `AddShare` copies the flag verbatim (`lifecycle.go:548`) and nothing defaults it.
2. **Step 6** — the `volume_test.go` fence updated to match the working tree exactly: `Enabled: true` in the fixture, the `bytes.Equal` Handle assertion, the whole new `TestRuntimeExporter_DisabledShareIsRefusedAndUnlisted` function, and a closing note that the file's imports must include `"bytes"` and `"errors"`.

Fence/working-tree sync verified by extraction rather than by eye: pulled the Step-6 ```` ```go ```` fence out of the plan and diffed everything from `func TestRuntimeExporter_ResolvesVolumeFile` to end of fence against the same range of `internal/adapter/nbd/volume_test.go` → `FENCE_BODY_IDENTICAL`. The plan's Step-1 `handshake_test.go` fence was re-checked after the controller's `283105a1` errcheck fix and still matches the file → `HANDSHAKE_FENCE_IN_SYNC`. Plan fence count is even (92), so no fence was broken by the edit.

## Commands run

```
go build ./...                                                   # ok
go vet ./internal/adapter/nbd/                                   # clean
gofmt -l internal/adapter/nbd/                                   # empty
go test ./internal/adapter/nbd/... -race -count=1                 # ok, 3.466s
go test ./internal/adapter/nbd/ -v -count=1                       # 21 pass, 0 fail
GOTOOLCHAIN=go1.26.0 golangci-lint run ./internal/adapter/nbd/...  # 0 issues
bash test/no-ai-attribution.sh c13b595f                           # no AI attribution in 3 commit(s)
git add internal/adapter/nbd/volume.go internal/adapter/nbd/volume_test.go && git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "fix(nbd): refuse and omit a disabled share at export resolution"
git add docs/.../2026-10-08-dittofs-nbd-block-protocol.md && git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "docs(nbd): plan the disabled-share gate and the file-handle assertion"
```

## Concerns

1. **The `Enabled`-defaulting trap is wider than this task.** `runtime.ShareConfig.Enabled` has no default, and `AddShare` copies it verbatim, so every fixture that omits it builds a disabled share. It was harmless until this gate landed. Task 3's adapter tests and Task 4's connection tests construct shares the same way and will need `Enabled: true` — worth a line in the ledger so it is not rediscovered as a mystery `ErrUnknownExport`.
2. **`List` now calls `GetShare` per share** where it previously only called `GetRootHandle`. `GetShare` returns a snapshot copy under the share-service read lock (`pkg/controlplane/runtime/shares/service.go:406-423`), so this is a locked read per share on a path that is already doing a metadata `Lookup` and a `ReadDirectory` per share. `NBD_OPT_LIST` is a handshake-time operation, so the cost is not on the hot path, but the review should know the extra call is there and that it is deliberate.
3. **The gate is enforced at resolution only.** A client that resolved an export *before* the share was disabled keeps its `*Export` and, through Task 4's write path, keeps writing — `PrepareWrite`/`CommitWrite` do not re-consult `Enabled`. This matches the other adapters, whose disabled-share checks are also at establishment (NFS MOUNT, SMB TREE_CONNECT, NFSv4 share-entry), and the `Share.Enabled` doc comment describes the flag as rejecting *new* clients. Closing the window for an already-connected NBD client would mean re-checking per command, which the brief does not ask for and which would put a share-service lock on the hot path. Flagging it as a scope question for the whole-branch review, not as a defect in this fix.
4. Deferred Minor findings were left alone as instructed: `List` coverage beyond the new test, the full-GO-reply-sequence test, the handshake read deadline, the unvalidated `NBD_FLAG_C_FIXED_NEWSTYLE` bit, `writeSimpleReply`'s `//nolint:unused`, and the over-long-option session close. `RepErrTooBig`'s session close is unchanged.
