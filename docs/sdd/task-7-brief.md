## Task 7: Differential oracle against `qemu-nbd`

**Files:**
- Create: `test/nbd/run-oracle.sh`, `test/nbd/README.md`,
  `internal/adapter/nbd/oracle_test.go`
- Modify: `internal/adapter/nbd/handlers_control.go` (+ its tests) for the FLUSH
  metadata gap below. This is the one functional change in Task 7; everything
  else is test scaffolding. It is here rather than in a new task because the
  oracle is what makes the durability path observable, and a gap found while
  building the instrument should be closed with it.
- Modify: nothing else. `test/required-tests.json` is deliberately **not** touched:
  `test/required-tests.sh` runs `go test` for every name in the manifest, and
  the Tier A oracle needs a QEMU install, so listing it there would make the
  required set fail on every machine without one. A QEMU-dependent test is a
  manual regression check, not a required one — the manifest's own purpose is
  the opposite: names whose removal must be loud. (If Tier A is ever split so
  the QEMU-free half can run alone, that half can be listed.)

### FLUSH commits the block store but not the metadata — found while auditing this task

`handleFlush` (`internal/adapter/nbd/handlers_control.go`) calls only
`h.commit` → `common.CommitBlockStore`, the **block store** half of the barrier.
The metadata half is missing, and the sibling protocols all do both:
`common.FlushStableWrite` (`internal/adapter/common/write_payload.go:201`) is
exactly `CommitBlockStore` **then** `metaSvc.FlushPendingWriteForFile`, and it is
what NFSv3 WRITE (`internal/adapter/nfs/v3/handlers/write.go:299`), NFSv4 WRITE
(`internal/adapter/nfs/v4/handlers/write.go:238`), NFSv3 COMMIT
(`internal/adapter/nfs/v3/handlers/commit.go:257-277`), NFSv4 COMMIT
(`internal/adapter/nfs/v4/handlers/commit.go:105-119`) and SMB CLOSE/FLUSH
(`internal/adapter/smb/handlers/close.go:242-272`) call. Neither the plan nor the
spec mentioned `FlushStableWrite` or `FlushPendingWriteForFile` anywhere —
`grep` returns nothing in both documents — so this is an omission in the design,
not an implementer error.

**Measured, not inferred.** With deferred commits on (the default:
`pkg/metadata/service.go:131` `s.deferredCommit.Store(true)`), after
`PrepareWrite` + `CommitWrite` + `h.commit` the raw metadata store's mtime is
**unchanged**; calling `FlushPendingWriteForFile(ctx, handle, false)` afterwards
returns `flushed=true` and **does** change it. So the pending size/mtime state
survives an NBD FLUSH that the client was told succeeded.

**Why this is Minor rather than Important, stated so the severity is auditable
rather than asserted:**
- **The size can never be wrong.** NBD's `handleWrite` calls
  `PrepareWrite(authCtx, exp.Handle, req.Offset+uint64(req.Length))`
  (`internal/adapter/nbd/handlers_write.go:95`) and refuses anything past
  `exp.Size` first (`:61`), so `MaxSize` ≤ the size already stored; nothing in the
  NBD adapter calls `SetFileAttributes` or any other size setter. The flush is
  also max-only (`pkg/metadata/io.go:474` `if size > file.Size`), so it cannot
  shrink the export either.
- **The mtime is not observable over NBD.** The dispatch has five arms —
  `CmdRead`, `CmdWrite`, `CmdFlush`, `CmdTrim`, `CmdWriteZeroes`
  (`internal/adapter/nbd/connection.go:261-281`) — and no metadata command, so a
  client cannot read the mtime back. It is also not consumed by the block layer
  or the GC.
- **The state is bounded and not lost.** `RecordWrite` keeps one state per
  handle (`pkg/metadata/pending_writes.go:83-92`), so the pending set is bounded
  by the number of volume files rather than by write count, and the shutdown path
  flushes it.

**RULING: add the metadata flush.** `commit` becomes `FlushStableWrite`, so
NBD's barrier is the same barrier the other adapters use. The alternative —
keeping `CommitBlockStore` alone and documenting why NBD is different — was
rejected because NBD is not different in any way that matters here: an NBD
device is a boot path (Task 8 boots a kernel from it, and a kernel issues
`FLUSH`), the command's own spec text is an unconditional durability barrier,
and the cost of the fix is one type widening plus one body change. The
omit-and-document option would have to argue that a client told "durable" need
not actually be durable, which is exactly the class of bug the residency-truth
gate (spec §7) exists to prevent.

What was *not* acceptable is the state before this ruling: the omission was
invisible. The code comment on `commit` says the durability rule *"has exactly
one implementation across the adapters instead of one per protocol"*, which was
true of the block-store half and silent about the other half.

**The fix, and why the type widening is provably safe.**
`FlushStableWrite`'s body already passes `blockStore` to `CommitBlockStore`,
whose parameter is `Committer` (`internal/adapter/common/write_payload.go:125`),
so the concrete `*engine.Store` must already satisfy the interface — widening the
declared parameter changes no call site and no behaviour. NFS keeps passing a
concrete `*engine.Store`.

**Interfaces:**
- Consumes: a DittoFS NBD server listening on a real TCP port. The script does not start one; it takes a port.
- Produces: `test/nbd/run-oracle.sh` — exits nonzero on any divergence.

This is the independent check on a hand-rolled codec. `qemu-nbd` is a known-good NBD server; the same client commands run against both, and the byte sequences and negotiated flags are diffed.

**The oracle needs a server on a real port, and nothing in this repo starts one.**
DittoFS serves adapters from the control plane, not from a config file:
`pkg/config/config.go:37` says *"Dynamic configuration (users, groups, shares,
stores, adapters) is managed through the REST API and stored in the control
plane database"*, and `pkg/adapter/smb/config.go`'s `Enabled` is set `true`
unconditionally by the adapter factory (`cmd/dfs/commands/start.go:967`), so
there is no "off" a config file could express. A live run therefore needs
`dfs start` plus `dfsctl store metadata add`, `dfsctl store block add`,
`dfsctl share create`, and `dfsctl adapter enable nbd` — the sequence
`test/scenarios/setup.sh:73-81` already performs for smb and nfs, needing only
an `nbd` line. (It does need one non-obvious config value: the NBD bind gate
refuses a wildcard bind without `allow_remote_bind`, and the adapter factory
never sets `BindAddress`, so the adapter must be created with
`dfsctl adapter edit nbd --config '{"bind_address":"127.0.0.1"}'`.)

**But a live server cannot serve this oracle, for a reason that is not
incidental.** `RuntimeExporter.Resolve` (`internal/adapter/nbd/volume.go`)
resolves a volume by `Lookup`ing the share root, then `volumes/`, then the
volume name — and **no command in the repo creates a volume file**:
`grep -rn "volume" cmd/dfsctl/` returns nothing, and
`internal/adapter/nbd/volume.go:21` records that *"the operator, not the
protocol, decides who owns the volumes"*, so the operator creates
`volumes/<name>` through the share's own NFS/SMB mount. And the volume is a
**file inside a share**, so the byte-level oracle also needs an fs-backed local
tier, which is a second piece of production configuration rather than a test
fixture. A script that stood all of that up would be testing share
provisioning, not the codec, and would be unusable in CI.

**So the oracle is split, and the split is the point.**

- **Tier A — the codec, in-process, no QEMU and no provisioning.** A Go test
  that serves the real `Handler` on a real TCP listener and runs the real
  `qemu-io` against it, comparing against `qemu-nbd`. This is where the value
  is: `qemu-io` is a real independent NBD client, which is exactly what a
  hand-rolled codec needs to be checked against, and the whole codec is
  reachable without the control plane because `internal/adapter/nbd`'s
  `Handler` is built from an `Exporter`, a `StoreResolver`, a
  `*metadata.Service`, an `Identity` and `Timeouts` — all of which a test can
  supply. `pkg/adapter/nbd/adapter_test.go:88` already serves a real listener
  and `internal/adapter/nbd/handshake_test.go:50` already does a real TCP
  handshake, so neither the listener nor the wire path is new machinery.
  `test/nbd/run-oracle.sh` remains the driver: it checks QEMU is present, runs
  `go test` for Tier A, and reports.
- **Tier B — the deployed path, when someone has a server.** The same script
  accepts an already-running DittoFS NBD port via an environment variable or a
  flag, and skips Tier B with a printed reason when it is absent. It must not
  *discover* a server or silently fall back to Tier A: the script's output has
  to say which tier ran, or a green run means nothing in particular.

The flag-parity check (Step 2) belongs to **both** tiers, because it needs only
a handshake and no data path.

- [ ] **Step 0: Close the FLUSH metadata gap (its own commit)**

Do this first and commit it alone, so the functional change is reviewable
independently of the test scaffolding that follows.

1. Widen `FlushStableWrite`'s `blockStore` parameter from `*engine.Store` to
   `common.Committer` (`internal/adapter/common/write_payload.go:79-84`).
   Type-only; the body uses `CommitBlockStore`, which already takes `Committer`.
2. In `internal/adapter/nbd/handlers_control.go`, change **`commit`** — not
   `handleFlush` alone — to call
   `common.FlushStableWrite(h.authContext(ctx, exp), h.meta, bs, exp.Handle, exp.PayloadID, true)`
   in place of `common.CommitBlockStore(ctx, bs, exp.PayloadID)`. `commit` is
   already the single barrier both call sites share, so changing it there fixes
   FLUSH *and* FUA with one edit, and keeps the "one implementation" property
   the code comment claims. Building the `AuthContext` inside `commit` from the
   `ctx` it already receives is correct for both callers: `handleWrite` passes
   the same command context it used for `PrepareWrite`, and
   `h.authContext(ctx, exp)` is pure.

   Then correct `commit`'s doc comment, which currently says the durability rule
   *"has exactly one implementation across the adapters instead of one per
   protocol"* while implementing only the block-store half — the sentence
   becomes true once `FlushStableWrite` is the body, and the
   `decision:` paragraph about serialization (which is what makes a FLUSH a
   barrier over already-acked writes) still holds and should stay.
3. Add a test that reads the raw store before and after a FLUSH and requires the
   mtime to move. Build the fixture with `newIntegrationFixtureAs(t, integrationIdentity)`
   — with the wrong identity `PrepareWrite` fails `AccessDenied` and the test
   never reaches the barrier.

**This breaks the four `connection_test.go` FLUSH tests, and the failure is a
panic rather than an assertion.** `newTestHandler` (`connection_test.go:177`)
passes `nil` for the metadata service, and `FlushPendingWriteForFile` dereferences
it: measured, a nil `*metadata.Service` panics with `runtime error: invalid
memory address or nil pointer dereference`. The four affected tests are
`TestConnection_FlushIsAckedWhenDurable` (`:1119`),
`TestConnection_FlushIsEIOWhenNotDurable` (`:1139`),
`TestConnection_FlushWithNonZeroLengthIsEINVAL` (`:1161`) and
`TestConnection_FlushOnReadOnlyExportIsPermitted` (`:1177`). Move them onto the
real fixture (`newWriteVolumeFixture`, whose `newHandler` passes
`f.rt.GetMetadataService()`), as Tasks 5 and 6 already did for the WRITE and
punch paths. The two that assert on `st.flushes` still work, because `storeFake`
is passed through `resolverFor`; the EINVAL case must additionally keep asserting
`st.flushes == 0`, which is what proves a malformed FLUSH never reaches the
barrier at all.

Expected: the new test fails before step 2 and passes after. If it passes before,
it is not testing the gap — check that `deferredCommit` is on (it is by default)
and that the read is from `rt.GetMetadataStoreForShare(integrationShare)`, not
from the service's cached path.

```bash
git add internal/adapter/common/write_payload.go internal/adapter/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S \
  -m "fix(nbd): commit the metadata half of the FLUSH barrier"
```

- [ ] **Step 1: Write the script**

`test/nbd/run-oracle.sh`, following the existing `test/crash/*.sh` convention: a `#!/usr/bin/env bash` header with a comment block explaining what it proves, `set -euo pipefail`, a usage line, and a prerequisites check.

**Tier A is a Go test, so the script's job is narrow.**
`internal/adapter/nbd/oracle_test.go` (package `nbd`, internal — see the note below on why) does this:

1. Skip, not fail, when `qemu-nbd` or `qemu-io` is absent: `t.Skip` with the
   missing binary named, so `go test ./...` stays green on a machine without
   QEMU while the script — which checks first — reports the skip as a failure.
2. Build the same fixture Tasks 4–6 use: `testruntime.NewWithShareStore`, a
   metadata store, a share with `Enabled: true`, and the volume file. The store
   is an in-memory fake of the seven-method `BlockStore`, so the oracle
   compares **the codec and the handshake**, not the block layer — state that
   in the test's doc comment, because a reader who believes it covers
   `PunchHole` will trust it for something it never tested.
3. `net.Listen("tcp", "127.0.0.1:0")` and serve the real `Handler` on it with a
   real accept loop — `go NewConn(h, c).Serve(ctx)` per accepted connection —
   exporting the fixture's export name and taking the port from the listener,
   not from config (`BaseAdapter.Port()` returns the *configured* port,
   `pkg/adapter/base.go:598`). **Not `pipe(t)` and not `go h.serve(...)`:** the
   client is `qemu-io`, a separate process, so an in-process socket pair cannot
   be reached. `pipe`/`serveConn` remain the right tools for every other test in
   the package, which is why this is easy to get wrong by habit.
4. Start `qemu-nbd` as a subprocess against a `t.TempDir()` scratch file **of exactly the export's size**, with `-f raw -p <free port> -b 127.0.0.1 --persistent
   --discard=unmap -x export/boot`, and stop it in `t.Cleanup`. **`-x export/boot` is mandatory** — measured, `qemu-nbd` exposes an *empty* export name by default, so the `nbd://…/export/boot` URI that works against DittoFS fails against the oracle with `server reported: export 'export/boot' not present`. Without `-x`, every command diverges for a reason that has nothing to do with the codec.

   **The scratch file must be the export's size, and the export must be large enough for the sequence below.** The two sizes have to agree, because every command in Step 5 is bounds-checked against the export: DittoFS refuses a WRITE past the end with `NBD_ENOSPC` and a READ or TRIM past the end with `NBD_EINVAL`, while `qemu-nbd` against a larger file would succeed. A mismatch therefore shows up as a transcript divergence that names neither cause. The committed `newWriteVolumeFixture` builds a **1 MiB** volume (`internal/adapter/nbd/integration_test.go:142` sets `Size: 1 << 20`) and the sequence below reaches **9 MiB**, so Tier A cannot use that fixture's volume as-is: it needs its own fixture sized to the sequence, e.g. 16 MiB, built by the same helper with the size parameterised (`newIntegrationFixtureAs` already takes the identity; add the size beside it rather than writing a second provisioning block).
5. Run the identical `qemu-io -f raw -c '<cmd>'` sequence against both, capture
   stdout+stderr **and each command's exit status**, normalize the timing line,
   and require the results to match. Addresses are
   `nbd://127.0.0.1:<port>/export/boot` — see the export-name note below.

   **The transcript must be normalized before it is diffed, and the diff must
   include exit codes. Both halves are measured requirements, not hygiene:**
   - **Normalize.** `qemu-io` prints a timing line after every command
     (`1 MiB, 1 ops; 00.00 sec (1.533 GiB/sec and 1569.8587 ops/sec)`) whose
     numbers change on every run. Measured: two *identical* runs against the
     *same* `qemu-nbd` produced a diff on that line alone, so a literal
     line-for-line comparison can never pass. Drop every line containing
     `ops;` before diffing (`sed '/ops;/d'`), which was measured to make two
     identical runs byte-identical.
   - **Record exit codes.** `qemu-io` reports a pattern mismatch on **stdout**
     and exits **1**; the failure line is
     `Pattern verification failed at offset 0, 1048576 bytes`. Measured, and
     the exit code is the load-bearing half: a read of the wrong pattern still
     prints `read 1048576/1048576 bytes at offset 0` and a timing line, so a
     transcript that ignores the status reads as success. Emit `exit=$?` after
     each command into the transcript, and run the sequence with `set +e` so
     one failure does not truncate the comparison.

   This is also what makes the `discard` step meaningful: measured on
   `qemu-nbd` 11.1.2, `read -P 0` after a discarded range exits **0** with
   `--discard=unmap` and exits **1** without it. The exit code is the entire
   signal.

`test/nbd/run-oracle.sh` then only has to: refuse without `qemu-nbd`,
`qemu-img` and `qemu-io` (naming the missing binary); run
`go test ./internal/adapter/nbd/ -run TestOracle_ -v -count=1` and fail if it
skips; run the Python flag-parity handshake against both servers; and, when
`DITTOFS_NBD_PORT` is set, additionally drive Tier B. It must print which tier
ran — a green run that does not say what it checked means nothing.

**Exactly what Tier A constructs, because the pieces are not all reusable.**

```go
// Package nbd, NOT nbd_test, and that is a deliberate choice with a verified
// basis rather than an inheritance.
//
// The reason to be in-package is that this test needs pieces Tasks 4-6 already
// built: storeFake (a seven-method BlockStore fake whose PunchHole zeroes the
// range and whose ReadAt zero-fills past the written extent), resolverFor, and
// integrationIdentity. An external test package cannot see any of them and
// would have to redefine every one -- including a second BlockStore fake that
// would drift from the one the other tests use.
//
// It does NOT need pipe, sendRequest, readReplyHeader, completeHandshake or
// exporterFor: the client here is qemu-io, not a test goroutine, so the
// handshake and the framing are the client's business.
//
// The plan originally specified package nbd_test on the grounds that an
// in-package test cannot import internal/adapter/testruntime. That is FALSE:
// integration_test.go is package nbd and imports it at line 10, and
// `go test ./internal/adapter/nbd/ -run TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO`
// compiles and passes. There is no cycle either way -- `go list -deps
// ./internal/adapter/testruntime` contains no entry for dittofs/internal/adapter/nbd.
// So the constraint was imaginary, and in-package is strictly better here.
//
// What this test does NOT reuse is the fixture's VOLUME SIZE: the sequence
// below reaches 9 MiB and the committed fixture builds a 1 MiB volume, so Tier
// A provisions its own volume at the size the sequence needs. Add the sized
// helper BESIDE the existing one rather than changing its signature, which four
// other tests call:
//
//   func newIntegrationFixtureAs(t *testing.T, id Identity) (*runtime.Runtime, string, *engine.Store) {
//       return newIntegrationFixtureAsSized(t, id, 1<<20)   // unchanged behaviour
//   }
//   func newIntegrationFixtureAsSized(t *testing.T, id Identity, size uint64) (*runtime.Runtime, string, *engine.Store) {
//       // ... the existing body, with `Size: ptr(size)` in SetFileAttributes
//   }
rt, name, _ := newIntegrationFixtureAsSized(t, integrationIdentity, 16<<20)
exp, err := NewRuntimeExporter(rt, integrationIdentity).Resolve(context.Background(), name)

// The block layer is the in-package fake. It needs no sizing: ReadAt
// zero-fills both past the written extent and past `len(data[id])`, so the
// never-written-range step below compares equal without any setup.
st := newStoreFake()

// The server must be on a REAL TCP PORT, because the client is `qemu-io` -- a
// separate process. `pipe(t)` and `go h.serve(ctx, server)` are the right tools
// for every other test in this package and the WRONG ones here: pipe's pair is
// an in-process loopback socket, so an external process can never reach it.
ln, err := net.Listen("tcp", "127.0.0.1:0")
port := ln.Addr().(*net.TCPAddr).Port   // from the listener, not from config:
                                        // BaseAdapter.Port() is the CONFIGURED
                                        // port (pkg/adapter/base.go:598).

// The exporter must be the REAL one, for a reason that is not stylistic --
// see the export-name paragraph below.
h := NewHandler(NewRuntimeExporter(rt, integrationIdentity), resolverFor(st),
    rt.GetMetadataService(), integrationIdentity,
    Timeouts{Read: 30 * time.Second, Write: 30 * time.Second, Idle: 5 * time.Minute},
    func() bool { return false })

ctx, cancel := context.WithCancel(context.Background())
t.Cleanup(func() {
    cancel()
    _ = ln.Close() // unblocks Accept; the loop then returns
})
go func() {
    for {
        c, err := ln.Accept()
        if err != nil {
            return // listener closed by cleanup
        }
        go NewConn(h, c).Serve(ctx)
    }
}()

// qemu-io addresses it as nbd://127.0.0.1:<port>/export/boot -- the ONE-slash
// form, which the client turns into the wire name `export/boot`. That is why
// the real exporter is required above; see the paragraph after this fence.
```

**The exporter must be the real `NewRuntimeExporter`, NOT `exporterFor(exp)`,
and this is a correctness requirement rather than a preference.** `qemu-io`
sends the export name with one leading `/` stripped — `nbd://127.0.0.1:<port>/export/boot`
becomes the wire name `export/boot` (see the export-name paragraph below).
`exporterFor` builds a `fakeExporter`, whose `Resolve` is an **exact map
lookup** keyed on `testExportName` = `"/export/boot"` (`handshake_test.go:18-23`,
`connection_test.go:135`). Measured: `fakeExporter.Resolve(ctx, "export/boot")`
returns `nbd: unknown export` while `Resolve(ctx, "/export/boot")` succeeds. So
a Tier A built on `exporterFor` would fail the handshake on the very first
`qemu-io` command with an unknown-export error that has nothing to do with the
codec — and it would look like a codec bug. The real exporter folds the share
half through `metadata.NormalizeShareName` (`pkg/metadata/validation.go:176`),
which re-prepends the slash, so it accepts the spelling the client actually
sends. This is exactly the fold the export-name paragraph pins, and it is worth
having the oracle exercise it rather than fake around it.

The metadata service is real and unavoidable: `handleTrim` and
`handleWriteZeroes` call `h.meta.PunchHole`, which panics on a nil service —
the same reason Tasks 5 and 6 needed `writeVolumeFixture`. The block store stays
the `storeFake` (`resolverFor(st)`), so the oracle compares the codec and the
handshake against a fake block layer; `storeFake.PunchHole` does zero the range
(`connection_test.go:112-116`) and `ReadAt` zero-fills past the written extent,
which is what makes the `discard` and `write -z` steps comparable. Say this in
the test's doc comment, because a reader who believes it covers `PunchHole` will
trust it for something it never tested.

What it does:

1. Refuse to run without `qemu-nbd`, `qemu-img`, and `qemu-io`; name the missing binary in the error.
2. Create a scratch file **of exactly the export's size** (see the construction note above: a size mismatch turns into a transcript divergence that names neither cause) and serve it with
   `qemu-nbd -f raw -p <oracle-port> -b 127.0.0.1 --persistent --discard=unmap -x export/boot`
   in the background.

   **`-x export/boot` is not optional either, and its omission is the more
   confusing of the two.** Measured: `qemu-nbd` exposes an **empty** export name
   by default, so `nbd://127.0.0.1:<port>/export/boot` — the URI that works
   against DittoFS — is refused by the oracle with
   `server reported: export 'export/boot' not present` on every single command.
   The oracle needs the flag even though the code under test does not — the
   asymmetry is what makes it easy to omit.

   **`--discard=unmap` is not optional, and leaving it out makes the oracle lie.**
   `qemu-nbd` starts with `int flags = BDRV_O_RDWR` (`qemu-nbd.c:589`) and
   `BDRV_O_UNMAP` is a separate bit (`include/block/block-common.h:246`,
   *"execute guest UNMAP/TRIM operations"*) that only `--discard=unmap` sets —
   `bdrv_parse_discard_flags` clears it and re-sets it only for `on`/`unmap`
   (`block.c:1154`). Without it a `NBD_CMD_TRIM` reaches the block layer and is
   dropped: qemu-nbd replies **success** and the data is still there. Since
   DittoFS's `PunchHole` genuinely zeroes the range, the `discard` step below
   would then diverge on the *oracle's* configuration and be reported as a
   codec bug — the most expensive kind of false positive, because the natural
   fix is to make DittoFS stop zeroing. A server that drops a discard is legal
   (*"A server MAY discard length bytes starting at offset, but is not required
   to"*), so this is not a qemu-nbd defect; it is a configuration the comparison
   has to pin.

   `-b 127.0.0.1` is written explicitly rather than relied on: the oracle is a
   throwaway server holding a scratch file, and it has no business listening on
   anything but loopback. `--persistent` (`-t`, *"don't exit on the last
   connection"*) keeps it alive across the several `qemu-io` invocations.
3. Against `qemu-nbd` and against DittoFS, run the same `qemu-io` command sequence and capture both transcripts:
   - `qemu-io -f raw -c 'write -P 0x11 0 1M'`
   - `qemu-io -f raw -c 'write -P 0x22 4M 64k'`
   - `qemu-io -f raw -c 'read -P 0x11 0 1M'`
   - `qemu-io -f raw -c 'discard 1M 1M'` then `read -P 0 1M 1M`
   - `qemu-io -f raw -c 'write -z 8M 1M'` then `read -P 0 8M 1M`
   - a read of a never-written range
   - `qemu-io -f raw -c 'flush'`
4. `qemu-img info` against both, and diff the reported size and format. Measured deterministic — the output contains no timing and two runs against the same server are byte-identical, so this one is a plain diff.
5. Diff the two normalized transcripts (timing lines dropped, exit codes included). Any difference is a failure, with both transcripts printed.

**Export-name spelling, and why it is worth pinning.** QEMU turns a URI path into
the export name by stripping exactly one leading `/` (`nbd_parse_uri`,
`qemu-nbd.c:1563-1569`: `p = g_uri_get_path(uri); if (p[0] == '/') { p++; }`). So
`nbd://127.0.0.1:<port>/export/boot` sends the export name `export/boot`. That
resolves against DittoFS because `RuntimeExporter.Resolve` folds the share half
with `metadata.NormalizeShareName` (`pkg/metadata/validation.go:176`), which
re-prepends the slash — so both `export/boot` and `/export/boot` reach the same
volume. Write the single-slash form in the script and say in a comment that the
fold is what makes it work, because a reader who sees the volume listed as
`/export/boot` will otherwise "fix" the URI to `//export/boot` and get the same
result for a different reason.

**And the oracle has to be told to answer to that name.** Measured: `qemu-nbd`
exposes an **empty** export name unless given `-x`/`--export-name` — with no
`-x`, `nbd://127.0.0.1:<port>/x` fails with `server reported: export 'x' not
present`, and even the pathless form works only because it resolves to the empty
name. `-x export/boot` makes the two servers answer to the same name, which is
what makes one URI string usable against both. The asymmetry is easy to miss
because it is the *oracle* that needs the flag, not the code under test.

- [ ] **Step 2: Add the flag-parity check**

The oracle's most valuable assertion is not the data path — it is the flag set:

```bash
# qemu-nbd advertises its own flag set. Ours is deliberately narrower (no
# CAN_MULTI_CONN, no SEND_CACHE, no SEND_DF). What must match is the flags we
# DO advertise, because those are the ones a client acts on without verifying.
```

Capture the negotiated transmission flags for both servers and assert that every flag DittoFS advertises is also advertised by `qemu-nbd`, and specifically that `SEND_FLUSH` and `SEND_FUA` are present on both. `libnbd`'s `nbdinfo --list` does this in one command but **is not in Homebrew core** (`brew search --formula libnbd` returns only fuzzy matches; `nbdkit` likewise), so it cannot be the only path. The portable path is a ~40-line Python handshake in the script — read the greeting, send `NBD_FLAG_C_FIXED_NEWSTYLE`, send `NBD_OPT_GO` for the export, and decode the `NBD_INFO_EXPORT` reply's 16-bit flags — which needs nothing but a Python 3 interpreter. **Use the Python path as the primary**, because a script that silently degrades when a tool is missing is a script whose check nobody notices has stopped running. If `nbdinfo` is present, run it too and print its output for the record; do not make the assertion depend on it.

**The handshake has three field-layout traps, all hit and fixed while measuring this against `qemu-nbd` 11.1.2. Do not write it from memory:**
1. **The greeting is 18 bytes: `NBDMAGIC` (8) + `IHAVEOPT` (8) + `gflags` (2, a `uint16`).** There is **no version field** in the fixed-newstyle greeting — the server sends those 18 bytes and then waits for the client's 4-byte flags. Reading `>QII` (16 bytes, as the old-style handshake suggests) or trying to read a version after the 18 desynchronizes the stream, and the *next* read hangs until the socket times out rather than failing with a clear error. This is the trap that cost the most time.
2. **`gflags` is a capability bitmap, not a version.** Check `gflags & NBD_FLAG_FIXED_NEWSTYLE` (bit 0) rather than comparing it to a number. (The version comparison belongs to the *old-style* handshake, which this is not.)
3. **`NBD_INFO_EXPORT`'s body is `u16 info_type, u64 size, u16 flags`** — decode it as `struct.unpack(">QH", body[2:12])`. Unpacking it as `">HQQ"` fails with `struct.error: unpack requires a buffer of 18 bytes`.

Measured reference values from `qemu-nbd` 11.1.2 against a 16 MiB raw file with `-x export/boot --discard=unmap`: `flags = 0x0c6d` — `HAS_FLAGS`(0), `SEND_FLUSH`(2), `SEND_FUA`(3), `SEND_TRIM`(5), `SEND_WRITE_ZEROES`(6), `SEND_CACHE`(10), `SEND_FAST_ZERO`(11). DittoFS advertises `FlagHASFlags | FlagSendFlush | FlagSendFUA | FlagSendTrim | FlagSendWriteZeroes` (`internal/adapter/nbd/handshake.go:101`) = **`0x006d`**, and every one of those bits is present in `0x0c6d` (verified by masking: `0x006d & 0x0c6d == 0x006d`). The two extra bits the oracle sets are `0xc00` = `SEND_CACHE | SEND_FAST_ZERO`, which DittoFS deliberately does not advertise — so the assertion must be **one-directional** (ours ⊆ theirs), never equality.

**A handshake that was actually run against `qemu-nbd` 11.1.2 and returned `0x0c6d`, written with all four traps fixed.** Use this rather than re-deriving it; the traps above cost four debug cycles to find:

```python
#!/usr/bin/env python3
"""Negotiate one export and print its transmission flags.

Usage: nbd-flags.py <port> <export-name>
Exits non-zero if the server does not offer fixed-newstyle or refuses the export.
"""
import socket, struct, sys

NBDMAGIC = 0x4e42444d41474943
IHAVEOPT = 0x49484156454f5054
NBD_FLAG_FIXED_NEWSTYLE = 1 << 0
NBD_FLAG_NO_ZEROES      = 1 << 1
NBD_OPT_GO   = 7
NBD_REP_ACK  = 1
NBD_REP_INFO = 3
NBD_INFO_EXPORT = 0

NAMES = {0: "HAS_FLAGS", 1: "READ_ONLY", 2: "SEND_FLUSH", 3: "SEND_FUA",
         4: "ROTATIONAL", 5: "SEND_TRIM", 6: "SEND_WRITE_ZEROES", 7: "SEND_DF",
         8: "CAN_MULTI_CONN", 9: "SEND_RESIZE", 10: "SEND_CACHE",
         11: "SEND_FAST_ZERO", 12: "BLOCK_STATUS_PAYLOAD", 13: "SEND_BLOCK_STATUS",
         14: "CAN_META_CONTEXT"}

def recvn(s, n):
    b = b""
    while len(b) < n:
        c = s.recv(n - len(b))
        if not c:
            raise EOFError("short read")
        b += c
    return b

def negotiate(port, export):
    s = socket.create_connection(("127.0.0.1", port), timeout=5)
    # TRAP 1: the greeting is 18 bytes -- NBDMAGIC(8) IHAVEOPT(8) gflags(u16).
    # There is NO version field in the fixed-newstyle handshake: the server sends
    # these 18 bytes and then waits for the client flags. Reading 16 bytes (or
    # reading a version after the 18) desynchronizes the stream, and the next
    # read hangs until the socket times out.
    magic, ihav, gflags = struct.unpack(">QQH", recvn(s, 18))
    if magic != NBDMAGIC:
        raise ValueError("bad NBDMAGIC")
    if ihav != IHAVEOPT:
        raise ValueError("bad IHAVEOPT")
    # TRAP 2: gflags is a capability bitmap -- test the bit, do not compare it
    # to a version number. (The old-style handshake's version field is the thing
    # people reach for here, and it is not on the wire.)
    if not (gflags & NBD_FLAG_FIXED_NEWSTYLE):
        raise ValueError(f"server lacks fixed newstyle (gflags=0x{gflags:04x})")
    s.sendall(struct.pack(">I", NBD_FLAG_FIXED_NEWSTYLE | NBD_FLAG_NO_ZEROES))
    name = export.encode()
    payload = struct.pack(">I", len(name)) + name + struct.pack(">H", 0)
    s.sendall(struct.pack(">QII", IHAVEOPT, NBD_OPT_GO, len(payload)) + payload)
    flags = None
    while True:
        _, _, rtype, rlen = struct.unpack(">QIII", recvn(s, 20))
        body = recvn(s, rlen)
        if rtype == NBD_REP_INFO and rlen >= 12:
            info_type = struct.unpack(">H", body[:2])[0]
            if info_type == NBD_INFO_EXPORT:
                # TRAP 3: u16 info_type, u64 size, u16 flags.
                size, flags = struct.unpack(">QH", body[2:12])
        elif rtype == NBD_REP_ACK:
            break
        elif rtype & 0x80000000:
            raise ValueError(f"server refused the export (reply type {rtype})")
    s.close()
    if flags is None:
        raise ValueError("no NBD_INFO_EXPORT in the reply")
    return flags

if __name__ == "__main__":
    f = negotiate(int(sys.argv[1]), sys.argv[2])
    print(f"flags = 0x{f:04x}")
    for bit in range(16):
        if f & (1 << bit):
            print(f"  {bit:2d} {NAMES.get(bit, '?')}")
```

Against DittoFS the same script must print `0x006d`, and the parity assertion is then `(dittofs & qemu) == dittofs`. Note the script takes the export name as an argument precisely so the same code runs against both servers — and that it must be given `export/boot` for both, which is what the `-x` flag above arranges.

- [ ] **Step 3: Write `test/nbd/README.md`**

Prerequisites (`brew install qemu` on macOS; the Linux packages on a Linux host), what each of the three scripts proves, and — explicitly — that the oracle validates the *codec and flags* while the KVM harness (Task 8) validates the *boot path*. State that the oracle needs no root, no VM and no share provisioning, so it is the cheap regression check, and state which tier needs what: Tier A needs QEMU only; Tier B needs a running server with an NBD adapter and a provisioned volume, and is skipped without `DITTOFS_NBD_PORT`.

- [ ] **Step 4: Run the oracle**

Run: `test/nbd/run-oracle.sh`
Expected: exit 0, Tier A transcripts identical, flag parity asserted for both servers.

**QEMU was installed while auditing this task.** `brew install qemu` was run on this machine (it was absent), giving `qemu-nbd`, `qemu-io` and `qemu-img` at `/opt/homebrew/bin/` — `qemu-nbd --version` reports **11.1.2**, which is what the measurements in this task were taken against. So the prerequisite is now met and Step 4 can actually be run. On a fresh machine the command is `brew install qemu` (macOS) or the distro's `qemu-utils` package (Debian/Ubuntu).

Do not skip the step and do not write a script you never ran: report the missing prerequisite and say exactly which steps are unverified. A script whose failure path was never executed is a script that has never run.

If it diverges, the divergence is the finding: a hand-rolled codec's first real client is where a transposed field shows up. Diagnose against the spec before changing anything.

- [ ] **Step 5: Commit**

```bash
git add test/nbd/ internal/adapter/nbd/oracle_test.go
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "test(nbd): differential oracle against qemu-nbd"
```

---

