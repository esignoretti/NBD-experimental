# Task 7 report — differential oracle against qemu-nbd

Plan: `docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md`, Task 7.
Base commit: `bf95d39e`.

## Commits

| SHA | Subject | Files |
| --- | --- | --- |
| `89ab139c` | `fix(nbd): commit the metadata half of the FLUSH barrier` | 4 files, +94/−17 |
| `1985a7f7` | `test(nbd): differential oracle against qemu-nbd` | 4 files, +824/−1 |

Both SSH-signed (`BEGIN SSH SIGNATURE` present in both commit objects).

## Step 0 — the FLUSH barrier committed only half of itself

`internal/adapter/nbd/handlers_control.go:53` called `common.CommitBlockStore`,
which commits the **block store** and returns. With deferred commits on — the
default, `pkg/metadata/service.go:131` — a WRITE's size and mtime are held in
pending state and never reach the store; only `FlushPendingWriteForFile`
publishes them. So FLUSH answered success while the file's metadata stayed
unwritten: a client told "durable" whose size and mtime a restart could lose.

Fix: `return common.FlushStableWrite(h.authContext(ctx, exp), h.meta, bs,
exp.Handle, exp.PayloadID, true)`, with `internal/adapter/common/write_payload.go`
widened to a `Committer`.

Regression test `TestIntegration_FlushPersistsPendingMetadata`
(`internal/adapter/nbd/integration_test.go`). It reads the **raw store's** mtime
through `rt.GetMetadataStoreForShare`, not through the metadata service, because
the service merges pending state into its reads — a read through it would show
the write's mtime before any flush and the test would pass while proving nothing.
The middle assertion pins that distinction: the mtime must NOT move across the
WRITE, only across the FLUSH.

**Fail-before verified on the assertion, not on a build error.** Reverting the
`commit` body to `common.CommitBlockStore` produced exactly:

```
integration_test.go:475: the raw store's mtime did not move across a FLUSH:
2026-10-12 14:02:38.496738 +0200 CEST -> 2026-10-12 14:02:38.496738 +0200 CEST,
so the barrier committed the block store but not the file's pending metadata
```

Restoring the fix: `ok github.com/marmos91/dittofs/internal/adapter/nbd 1.030s`.

## Step 2 — the oracle

`internal/adapter/nbd/oracle_test.go`. **`package nbd`, not `nbd_test`** — the
plan's external-package constraint was imaginary. An in-package test needs
`storeFake`, `resolverFor` and `integrationIdentity`, which Tasks 4–6 already
built; an external package would have to redefine them, including a second
`BlockStore` fake that would drift. `integration_test.go` is itself `package nbd`
and imports `internal/adapter/testruntime`, and `go list -deps
./internal/adapter/testruntime` contains no entry for
`dittofs/internal/adapter/nbd`, so there is no cycle either way.

Two tests:

- `TestOracle_DifferentialAgainstQemuNBD` — the same `qemu-io` sequence against
  the real handler on a real loopback TCP listener and against `qemu-nbd`
  serving a raw scratch file of exactly the export's size (16 MiB; the sequence
  reaches 9 MiB). Transcripts compared after scrubbing timings and the URI.
- `TestOracle_TransmissionFlagsMatchQemuNBD` — ours ⊆ theirs.

Helper `newIntegrationFixtureAsSized` added beside `newIntegrationFixtureAs`,
which became a one-line delegate passing `1 << 20`. No behaviour change for the
four existing callers.

### Measured flag words

```
DittoFS = 0x006d, qemu-nbd = 0x0c6d, ours-only = 0x0000, theirs-only = 0x0c00
```

`theirs-only = 0x0c00` is `SEND_CACHE` (bit 10) and `SEND_FAST_ZERO` (bit 11) —
both deliberately absent from this adapter, which is why the assertion is
one-directional rather than equality.

### Two normalization bugs found and fixed while bringing it up

Both were oracle bugs, not codec divergences, and both are the kind that would
have been misread as a codec finding:

1. **The URI is echoed in the output.** `qemu-io` and `qemu-img` print the URI
   they connected to, and the two servers are on different ports, so every line
   differed for a reason unrelated to the protocol. The URI is now rewritten to
   `nbd://HOST:PORT/export/boot` on both sides.
2. **`qemu-io`'s timing line.** `1 MiB, 1 ops; 00.00 sec (1.533 GiB/sec and
   1569.8587 ops/sec)` changes on every run — measured, two identical runs
   against the same server differ on that line alone. Every line containing
   `ops;` is dropped.

After both fixes the transcripts are byte-identical.

## Driver

`test/nbd/run-oracle.sh` (executable). Refuses without QEMU, naming the missing
binary and the install command, exit 2. Runs Tier A, **treats `--- SKIP` as a
failure** (a green run that skipped the only differential check must not exit 0),
echoes the measured flag words, runs Tier B only when `DITTOFS_NBD_PORT` is set,
and always prints which tier ran.

`test/nbd/README.md` documents prerequisites, what each tier proves, and
explicitly that the oracle validates the *codec and flags* while the KVM harness
(Task 8) validates the *boot path*.

### Real output

```
== NBD differential oracle ==
tier A: in-process codec oracle (internal/adapter/nbd/oracle_test.go)
=== RUN   TestOracle_DifferentialAgainstQemuNBD
--- PASS: TestOracle_DifferentialAgainstQemuNBD (0.71s)
=== RUN   TestOracle_TransmissionFlagsMatchQemuNBD
    oracle_test.go:202: measured flag words: DittoFS = 0x006d, qemu-nbd = 0x0c6d, ours-only = 0x0000, theirs-only = 0x0c00
--- PASS: TestOracle_TransmissionFlagsMatchQemuNBD (0.22s)
ok  	github.com/marmos91/dittofs/internal/adapter/nbd	1.942s
tier A: PASS
tier B: SKIPPED (DITTOFS_NBD_PORT is unset; export it to run against a live server)
run-oracle.sh: PASS (tier A only)
```

exit 0.

## Verification

```
go build ./...                                                   ok
gofmt -l internal/adapter/nbd/ test/nbd/                         empty
go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/... -count=1
  ok internal/adapter/nbd 18.613s
  ok pkg/adapter/nbd 7.884s
  ok internal/adapter/common 9.492s
./test/required-tests.sh                                         every required test ran and passed
./test/nbd/run-oracle.sh                                         PASS (tier A only), exit 0
```

`gofmt -l .` repo-wide reports `pkg/block/gc/repair_test.go`. Verified
pre-existing at `bf95d39e` by stashing the working tree and re-running: it is not
touched by this task and was left alone.

## Not verified

- **Tier B is a handshake smoke test, not a transcript replay.** It confirms the
  port accepts a connection and returns a fixed-newstyle greeting; it does not
  yet drive the Tier A sequence against a live `dfs`. The script and the README
  both say so rather than implying coverage.
- **No live-server run.** `DITTOFS_NBD_PORT` was never set, so Tier B has never
  executed against a real server.
- **The oracle does not cover the storage path.** The block layer is
  `storeFake`, so `engine.Store.PunchHole`, the journal, the CAS and durability
  are outside it. The discard step compares because `storeFake.PunchHole` zeroes
  a byte range in a map, which is enough to make the transcript meaningful but is
  not evidence about the real `PunchHole`.
- **The boot path is not covered.** That is the KVM harness (Task 8).

## Files

- `internal/adapter/nbd/handlers_control.go` (Step 0)
- `internal/adapter/common/write_payload.go` (Step 0)
- `internal/adapter/nbd/connection_test.go` (Step 0)
- `internal/adapter/nbd/integration_test.go` (Step 0 + `newIntegrationFixtureAsSized`)
- `internal/adapter/nbd/oracle_test.go`
- `test/nbd/run-oracle.sh`
- `test/nbd/README.md`

`test/required-tests.json` was not touched.
