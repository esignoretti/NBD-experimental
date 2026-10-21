# NBD-experimental

An NBD (Network Block Device) adapter for [DittoFS](https://github.com/marmos91/dittofs), extracted for review and merge proposal.

**Status:** a working prototype, now pipelined. The protocol is implemented end to end, it has booted a real kernel from a real export, and it serves several commands per connection concurrently — depth 16 measures 1.45× what the serialized dispatcher reached. It is still not production-ready: no CI job runs the kernel-boot rig, and a handful of minors are recorded but unfixed. Five defects found by review and by running the rig are fixed here. Never merged upstream. See [Status](#status) and [`docs/nbd-benchmarks.md`](docs/nbd-benchmarks.md).

**This is not a standalone project.** It is an extraction of one feature branch — 103 commits — from the DittoFS repository. Every file sits at its exact upstream path so that the commits apply 1:1 to DittoFS. See [Merging upstream](#merging-upstream).

**What is here and what is not.** The 53 files this branch touches are extracted verbatim; the rest of DittoFS is not included, so this repository does not build on its own and has no `go.mod`. That is deliberate — it is a patch set with documentation, not a fork. `docs/sdd/` is the exception: those artifacts were never tracked upstream (they lived in `.superpowers/`, which DittoFS gitignores), so they are added here under `docs/sdd/` and would not be part of a merge.

---

## What it does

DittoFS is a distributed filesystem that exposes shares over NFS and SMB. This branch adds a third protocol: **NBD**, which exports a file inside a share as a raw block device.

That means a VM can boot from a disk image stored in DittoFS, or a database can put its data files on one, without a filesystem in between. The bytes are the same bytes NFS and SMB serve — a volume is an ordinary regular file at `<share>/volumes/<volume>`, so you can independently verify a block write by reading the file over NFS.

```
NBD client  ──►  internal/adapter/nbd/  ──►  block engine  ──►  journal + CAS
 (qemu-nbd,        wire codec, handshake,     (ReadAt,           (chunking,
  nbd-client,       dispatch, handlers)        WriteAt,           durability)
  Linux kernel)                                PunchHole, Flush)
```

### What is implemented

| Feature | State |
|---|---|
| Wire codec — request/reply/option headers, magics, sizes | ✅ |
| Handshake — fixed-newstyle, `NBD_OPT_GO`/`INFO`/`EXPORT_NAME`/`LIST`/`ABORT` | ✅ |
| Adapter lifecycle, control-plane registration (port 10809) | ✅ |
| Bind gate — refuses a non-loopback bind without `allow_remote_bind` | ✅ |
| Healthcheck, degradation counter | ✅ |
| `READ`, `WRITE`, `FLUSH`, `TRIM`, `WRITE_ZEROES`, FUA | ✅ |
| Read-only exports | ✅ |
| `CAN_MULTI_CONN` — several connections, coherent across them | ✅ |
| Payload-length enforcement (advertised 32 MiB maximum) | ✅ |
| `BLOCK_STATUS`, structured replies, meta-context, `STARTTLS` | ❌ out of scope by design |

`CAN_MULTI_CONN` is advertised because the flag's condition holds structurally, not because it was configured: the wire layer holds no cache, and the block store is resolved per *share*, so every connection to one export reaches the same journal and the same per-file flush locks. A `FLUSH` or FUA on one connection is therefore a barrier over bytes another connection wrote. Two tests pin this — `TestIntegration_OneBlockStorePerShare` (every resolution returns the same `*engine.Store`) and `TestIntegration_FlushOnOneConnectionCoversAWriteOnAnother` (a write on one connection is visible through a `FLUSH` on a second) — and the second was verified to fail when the store is deliberately made per-connection.

`BLOCK_STATUS` and structured replies are deliberately absent, not unfinished: advertising a residency query whose under-report case is guarded only by a runtime invariant would trade a documented limitation for an undocumented data-loss path. [`docs/nbd-block-status.md`](docs/nbd-block-status.md) states what this costs (sparse copies become full copies), what would have to be true to lift it, and what to do instead. `STARTTLS` is refused rather than ignored — a client that believed its session was encrypted and was not has no way to notice.

### What was proven, and how

This is the part worth reading before trusting any of the above.

**A real kernel booted from a real NBD export.** Not a mock, not a loopback socket: `test/nbd/boot-qemu.sh` boots an Alpine `linux-virt` 6.6.142 kernel under QEMU (hvf on macOS, kvm on Linux), `mkfs.ext4` + mounts the NBD device, writes `hello.txt`, and a *second* VM reads it back. The recorded UUID `853580a8-d9e6-4789-ace8-94180888d0e2` in the rig's README is from that run.

**A differential oracle against `qemu-nbd`.** `test/nbd/run-oracle.sh` and `internal/adapter/nbd/oracle_test.go` drive the real handler with a real independent NBD client and compare wire behaviour. Measured transmission-flag words:

```
DittoFS  = 0x016d
qemu-nbd = 0x0d6d
ours-only    = 0x0000     (we advertise nothing qemu-nbd does not)
theirs-only  = 0x0c00     (SEND_CACHE and SEND_FAST_ZERO, which we deliberately do not set)
```

`qemu-nbd` is run with `--shared=0`. That is load-bearing: its `--shared` default is `1`, i.e. one client at a time, and it withholds `NBD_FLAG_CAN_MULTI_CONN` in that mode. Measured, `--shared=1` gives `0x0c6d` and `--shared=0` gives `0x0d6d` — the only difference is the bit under test, so comparing against the default would assert that this adapter behaves like a single-client server.

**A crash/reconnect mechanism proven end to end.** `test/nbd/crash-restart.sh` SIGKILLs the server mid-flight, proves the kernel client's I/O *pauses* rather than failing, then restarts the server inside the window and proves the read *resumes with the pre-crash bytes*. It passes, and its negative control fails in exactly the way it must. Getting there required an `nbd-client` built from upstream `master`: 3.26.1 has no `-T` at all, and 3.27.1's persist is silently dead on the netlink path (`persist_mode` is never assigned `true`). Transcripts in [`docs/nbd-benchmarks.md`](docs/nbd-benchmarks.md).

### Measured performance

Random 4 KiB read IOPS by queue depth, 10,000 operations, best of 3, against a RAM-backed store on a 2-vCPU Linux host:

| Depth | 1 | 2 | 4 | 8 | 16 | 32 | 64 |
|---|---|---|---|---|---|---|---|
| IOPS | 7,686 | 11,325 | **14,104** | 14,450 | 13,297 | 14,124 | 13,586 |

**Flat from depth 4.** That plateau is the finding that motivates the open work below: the dispatch loop handles one command at a time, so adding queue depth buys nothing past the point where a single command's latency is saturated. Run-to-run variance on this host is ±15–20%, so treat the shape as the finding and any single figure as approximate. Sequential reads are flat in depth too (1,259 → 1,302 MiB/s); sequential writes could not be measured reliably here and no figure is claimed.

**Chunk size was the single biggest win.** The default block-store profile chunks at 1 MiB, so a 4 KiB random read that misses cache fetches 1 MiB — up to 256× amplification. Setting `chunk_size: 131072` for VM images took depth-1 IOPS from 847 to 5,328 (**6.3×**). Reads never re-chunk, so this only affects newly written data.

All of this, plus the functional results, the crash/reconnect proof, the live-server investigation and the measurement traps that produce wrong numbers, is in **[`docs/nbd-benchmarks.md`](docs/nbd-benchmarks.md)**.

---

## Status

### Fixed in this repository

Two bugs found in a full code review and fixed on top of the branch. Both have regression tests that were verified to fail without the fix.

**P0 — a WRITE arriving during drain desynchronized the stream.** The draining branch answered `NBD_ESHUTDOWN` without consuming the payload the client had already sent, so the next header was parsed out of payload bytes. The connection then died on a later command with `NBD_EBADMAGIC` — the exact opposite of the branch's purpose, which is to pause the client for reconnect rather than error the device. Commit `89f2e4a`.

**P1 — the handshake was unbounded.** `ServeHandshake` armed no deadline, the base adapter sets none on an accepted socket, and the dispatch loop's first arm only runs after the handshake returns. A client could connect, send nothing, and hold a goroutine and a file descriptor forever — before naming an export, so before any authorization decision exists to fail. Fixed with a `Timeouts.Read` deadline plus a `ctx` poll in the option loop, so a shutdown also aborts a handshake in progress. Commit `69e80b8`.

### What this branch now does that it did not

**Pipelining — done, and measured.** A connection used to serve one command at a
time: a single loop read a header, handled the command and wrote the reply. That
is why the depth sweep was flat from depth 8 — the client's queue was a queue of
waits. Dispatch is now three roles (`internal/adapter/nbd/pipeline.go`): a reader
that is the only goroutine advancing the stream, a pool of workers that never touch
the connection, and a single writer that is the only one writing to it. The split
is not optional even at one worker — two readers would consume each other's
payload bytes, two writers would interleave a header and its payload.

| depth | before (serial) | after (`max_in_flight: 4`) |
|---|---|---|
| 1 | 22,075 | 21,141 |
| 8 | 39,682 | 51,282 |
| 16 | 40,160 | **58,139** |
| 32 | 39,215 | 54,945 |
| 64 | 38,167 | 51,020 |

Depth 1 is unchanged, which is the control: one command in flight is one command in
flight either way. Depth 16 moves 1.45×. The absolute numbers vary 15–20% run to
run on this host — the **shape** is the finding. A pool of eight measures within
noise of a pool of four, which is why four is the default.

The plan's premise was partly wrong and the results say so: it predicted
approaching the ~114,000 IOPS the engine reaches in-process, and the plateau
landed at ~55,000. The dispatcher was **a** ceiling, not **the** ceiling; the
store's admission control is next.

`max_in_flight` (default 4, clamped to 64) and `max_in_flight_bytes` (default
64 MiB) are configurable. The byte budget covers READ as well as WRITE, because
`handleRead` allocates on the worker — a WRITE-only budget would leave N concurrent
READs free to allocate N × 32 MiB with the counter untouched.

**FLUSH is a real barrier now.** Its correctness used to rest on serialization:
a command was handled and replied before the next was read, so every replied write
had been committed by the time a FLUSH was handled. A pool breaks that argument, so
the guarantee is explicit — a pending-write counter that FLUSH waits on, before it
is queued, on the reader goroutine.

### Defects found by running it, not by reading it

**A data race in `pkg/metadata`, reachable only now that dispatch is concurrent.**
`PendingWritesTracker.RecordWrite` returned a pointer into its own map, which
`deferredCommitWrite` read after the lock was released. That was safe only while
every protocol adapter dispatched one command at a time per file — exactly what NBD
no longer does. It now returns a copy taken under the lock.

**`SetPipelining` was never called.** The handler's fields were read by `serve` and
written by nobody: the adapter built the handler without them, so every configured
value was silently replaced by the default. A first benchmark A/B measured the
default against the default and reported that pipelining changed nothing — true of
that binary, false of the code.

**A cancelled context never reached the reader.** `ReadRequest` blocks in a syscall
that context cancellation does not interrupt, so shutdown waited out the idle
timeout — five minutes. The fix is a read deadline of *now*, not a budget: it is an
unblock, and a grace period would hold the connection open past the shutdown
window.

**And a worse one the above exposed:** replies were handed to the writer with a
plain `select` over `{send, ctx.Done()}`. When both are ready Go picks randomly,
and during a drain both always are — so roughly half the replies of a draining
connection were dropped by their own escape hatch and the client got `EOF` where it
should have got an answer.

### Open work

**1. `crash-restart.sh` needs a Linux host.** It requires root and an `nbd-client`
built from `master` — no released version can reconnect. Its first end-to-end run
found that it waited for the block *node* while `/sys/block/nbdN/size` was still 0,
so a write failed `ENOSPC` on a healthy export and the script blamed the adapter
identity. Fixed in `37f03113`.

It has since run end to end on a real Linux host against a 2 GiB export, and that
run found three more defects in the script, each of which had made a passing run
impossible or meaningless:

- **`-t` was never passed.** With one connection and no command timeout the kernel
  resets its timer instead of requeueing (`drivers/block/nbd.c:435` vs `:469`), so
  the request never reaches `wait_for_reconnect` — it hangs *past* the window
  rather than failing at it. `-T` alone cannot produce the pause the test claims.
  The script now requires a nonzero `--cmd-timeout` below `--dead-timeout`.
- **Pre-flight described the file, not the server.** A run pointed at a production
  share printed the production size and then crashed a fixture on another port;
  both runs looked identical until the numbers were compared. The script now reads
  `/sys/block/<dev>/size` after connecting and refuses on a mismatch, with
  `--expect-size` for servers whose metadata store is in memory.
- **Nothing proved the resumed read came from the server.** It is now cross-checked
  through a fresh client after the restart.

**2. One missing artifact.** "The deploy script" is referenced twice in
`test/nbd/deploy/config-server2.yaml` and does not exist. (`bench-fio.sh`, which
the pipelining plan cited for months, now exists.)

**3. The kernel-boot rig is not in CI.** The NBD unit tests and the differential
oracle now are — `.github/workflows/integration-tests.yml` installs `qemu-utils`
and runs `test/nbd/run-oracle.sh`, and four NBD gates are pinned in
`test/required-tests.json`. Booting a VM per pull request is a different cost
decision and has not been made.

**4. The live demo server is not usable for benchmarking.** The same depth sweep
against the live S3-backed host reads ~92 IOPS and stays flat, because its local
tier is not retaining reads (`hit-ratio: 0.40`, cache life expectancy 2.00 s at
p50). That is a cache-configuration finding, not an adapter one — but it needs its
own investigation before any number from that host means anything. Detail in
`docs/nbd-benchmarks.md` §5.

### Known minor issues

Every item the review recorded is now closed. What remains is one capability
deliberately refused and one performance opportunity, both explained rather than
fixed.

| Item | State |
|---|---|
| `max_message_size`, `default_export`, `uid`, `gid`, `timeouts.*` had tags but no reader | **fixed** — `uid`, `gid` and all four `timeouts.*` are read and range-checked; `max_message_size` and `default_export` are **deleted**, because the advertised maximum and the enforced maximum must be the same number and making only the enforcement configurable would be a conformance bug |
| UID/GID `0` unexpressible | **fixed** — `UIDSet`/`GIDSet` pointers, because every uint32 is a legal UID |
| `PunchHole` without a request deadline | **fixed** |
| `handleFlush` did not validate flags | **fixed** |
| No read-permission check on READ | **fixed**, and a short read is now `EIO` instead of being ignored |
| 10 dead constants, 6 test-only, 1 dead + 4 test-only codec functions | **deleted** |
| Export size was a handshake-time snapshot | **fixed** — re-resolved at most once per second (`Handler.sizeRefresh`), so a truncate over NFS/SMB becomes visible |
| `validate()` only checked the port | **fixed** — also refuses a remote bind with no connection cap, and negative pipelining values |
| No buffer pooling on READ/WRITE | **open** — the byte budget bounds the memory, but every command still allocates. A pool would need an ownership discipline the pipeline does not have yet |
| `NBD_CMD_BLOCK_STATUS` | **refused by design** — see [`docs/nbd-block-status.md`](docs/nbd-block-status.md) |


---

## Health

In the parent repository, at the branch tip:

```
go build ./...                                          OK
go vet ./...                                             clean
gofmt -l (both NBD packages)                             clean
go test -race -count=1 (both packages)                   ok 14.4s / ok 3.8s
go test -run Oracle                                      both PASS
go test -race ./internal/adapter/... ./pkg/adapter/... ./pkg/metadata/...
                                                         63 packages ok, 0 failures
```

**And re-verified as a merge into v0.36.0**, which is the state a reviewer would actually get — see [Merging upstream](#merging-upstream). `go build ./...` passes, both NBD packages pass, and 63 of 63 adapter and metadata packages pass.

**123 test functions and 5 benchmarks** across 16 test files: 4,949 lines of
tests against 2,768 lines of production code in `internal/adapter/nbd`, plus 502
against 457 in `pkg/adapter/nbd`. The rig under `test/nbd/` is another 2,257
lines of shell. (Before this branch's cleanup the split was 97 test functions
across 9 files, 3,758 test lines against 1,880 production — the production code
grew by the pipeline and the tests grew faster.)

---

## Layout

Every path below is where the file lives in DittoFS, unchanged.

```
internal/adapter/nbd/           7,717 LOC — the wire layer (2,768 prod + 4,949 test)
  protocol.go                     wire codec: magics, header encode/decode
  handshake.go                    greeting, option haggling, export resolution
  connection.go                   Conn, Handler, BlockStore; serve hands off to pipeline
  pipeline.go                     reader / worker pool / writer, byte budget, FLUSH barrier
  handlers_read.go                NBD_CMD_READ
  handlers_write.go               NBD_CMD_WRITE, TRIM, WRITE_ZEROES, payload drain
  handlers_control.go             NBD_CMD_FLUSH, the durability barrier
  volume.go                       RuntimeExporter: <share>/<volume> → Export
  payload_size.go                 the advertised-maximum bound
  reply.go                        reply writers
  reply_value.go                  the reply value handlers return instead of writing

pkg/adapter/nbd/                  959 LOC — the framework side (457 prod + 502 test)
  adapter.go                      BaseAdapter lifecycle, Serve, NewConnection
  config.go                       Config, defaults, validation
  healthcheck.go                  degraded-export reporting

test/nbd/                        the proof rigs
  README.md                       32 KB: topology, every command, every finding
  boot-qemu.sh                    40 KB: boots a real kernel from an export
  crash-restart.sh                36 KB: server-down/reconnect (needs Linux+root)
  bench-fio.sh                    depth sweep, sequential, fsync-per-4KiB
  fetch-artifacts.sh              20 KB: downloads the kernel + nbd.ko
  run-oracle.sh                   differential run against qemu-nbd
  deploy/config-server2.yaml      demo host config, with the memory budget

docs/nbd-benchmarks.md           ⚠ added here: every measured result, with method
                                 and the traps that produce wrong numbers
docs/superpowers/specs/          the design spec (§1–§13)
docs/superpowers/plans/          the 8-task implementation plan, and the pipelining plan
docs/sdd/                        ⚠ added here, not upstream: the SDD record —
                                 briefs, reports, review diffs, progress ledger
```

Everything above the `docs/sdd/` line exists in DittoFS at that path. `docs/sdd/` does not — see [What is here and what is not](#what-it-does). `docs/nbd-benchmarks.md` is also new here, and is deliberately *not* called `BENCHMARKS.md`: upstream has a 588-line `docs/BENCHMARKS.md` comparing DittoFS against JuiceFS and rclone, and on a case-insensitive filesystem (macOS default) the two paths would collide on merge.

### The files outside the adapter

The adapter is not entirely self-contained. These existing DittoFS files were
modified, and they are why a merge needs care:

| File | Change |
|---|---|
| `cmd/dfs/commands/start.go` | `createNBDAdapter` + `case "nbd"` registration |
| `pkg/controlplane/models/adapter.go` | `"nbd"` accepted as an adapter type |
| `internal/adapter/common/write_payload.go` | FLUSH barrier: commit the metadata half, not just the block store |
| `internal/adapter/common/normalize.go` | `ErrAccessDenied` → `EPERM` mapping |
| `internal/adapter/common/read_payload.go`, `payload_operation.go`, `doc.go` | shared helpers the NBD handlers reuse |
| `internal/adapter/nfs/v4/handlers/content_errmap_wiring_test.go` | test adjusted for the shared helper |
| `pkg/metadata/pending_writes.go` | **a real data race**, exposed by pipelining: `RecordWrite` returned a pointer into its own map, read after the lock was released |
| `.github/workflows/integration-tests.yml` | installs `qemu-utils` and runs the NBD oracle |
| `test/required-tests.json` | four NBD gates pinned |
| `.gitignore` | `test/nbd/artifacts/` (56 MB of downloaded kernel + modules) |

Two of these are genuine bug fixes that affect NFS and SMB too:

- **`write_payload.go`** — the FLUSH barrier committed only the block-store half,
  leaving a client told "durable" with metadata a restart could lose.
- **`pending_writes.go`** — `RecordWrite` returned a pointer into the tracker's own
  map, which `deferredCommitWrite` read after releasing the lock. That was safe
  only while every adapter dispatched one command at a time per file. NFS and SMB
  happen to satisfy that today, so the race was unreachable through them — which is
  why it was never observed. It is still a latent bug in the shared layer, and
  pipelining made it ordinary.


---

## Merging upstream

The point of this repository is to make a merge proposal possible, so the history was reconstructed to make one clean.

**All 53 files the branch touches are byte-identical to the original branch**, except one deliberate difference. Verified per file against `nbd-on-036`:

```
files in branch: 53
IDENTICAL: 52  DIFFERS: 1  MISSING: 0
  DIFF test/nbd/deploy/config-server2.yaml
```

The one difference is the redaction: that file carried the live deployment's public address and JWT secret. It is the only file that differs, and it differs on purpose.

(Whole-tree hashes necessarily differ — the original branch has 3,305 files and this extraction has the 53 the NBD work touches. What matters for a patch is that every one of those blobs is identical.)

**The history is 103 NBD commits** — the same commits, same authors, same timestamps, same messages, in the same order — replayed onto DittoFS **v0.36.0** (`51fc8443`). Unrelated upstream commits that the extraction would otherwise have carried are dropped; nothing about the NBD work is lost.

**The merge was tested, not assumed.** The result is not "the patches apply"; it is a tree that builds and passes at the base. Overlaying the branch's 53 files onto v0.36.0 and resolving one conflict:

```
go build ./...                                              PASS
go test ./internal/adapter/nbd/ ./pkg/adapter/nbd/          ok 8.2s / ok 1.7s
go test ./internal/adapter/... ./pkg/adapter/... ./pkg/metadata/...
                                                            63 packages ok, 0 failures
```

**The one conflict** is a rename collision. v0.36.0 has the lowercase `normalizeBlockStoreError(err)`; this branch exported it to `NormalizeBlockStoreError(err)` in `feat(nbd): connection loop and READ handler`. Take the branch's name — that is the whole resolution.

An earlier version of this section claimed the base was `f2183422` and that the range applied with two conflicts including `.gitignore`. Both were wrong, and the first was caught only by trying it:

```
internal/adapter/common/payload_operation.go:31:19:
    blockStore.WithPayloadScope undefined (type *engine.Store has no field or method WithPayloadScope)
```

`WithPayloadScope` arrives upstream in `2a3d7f76` (#2958) and is refined by `68946fe3` (#3030), both ancestors of v0.36.0 and both dropped by the extraction as unrelated. The `.gitignore` conflict disappeared once the extraction stopped carrying that file at all. The real base is **v0.36.0** (`51fc8443`).

`go test ./...` has two failures, neither from this work: `test/harness/known` fails at the v0.36.0 baseline too (macOS bash 3.2 has no `declare -A`), and `TestBLAKE3FasterThanSHA256` is a load-sensitive benchmark gate on an untouched package (`pkg/block` is not among the 53 changed files) that passes when run alone.

### What is in this repository but would not be part of a merge

Five files exist to document the extraction and would collide with DittoFS's own on a naive merge:

| File | Why it must not merge |
|---|---|
| `README.md` | DittoFS has its own; this one describes the extraction |
| `LICENSE` | DittoFS already has the identical MIT text |
| `.gitattributes` | pins line endings for this repo's contents |
| `docs/nbd-benchmarks.md` | measured results for this branch, written for a reviewer |
| `docs/nbd-block-status.md` | the BLOCK_STATUS refusal rationale, written for a reviewer |
| `docs/sdd/` (32 files) | the SDD record; lived in `.superpowers/`, which DittoFS gitignores |

Apply the other 53 files and nothing else. The verification above did exactly that: it overlaid 53 files and skipped these 37, and confirmed afterwards that all 53 were byte-identical to the branch with zero mismatches.

### To propose the merge

```bash
git clone https://github.com/marmos91/dittofs.git
cd dittofs
git checkout 51fc8443          # v0.36.0 — the verified base
git remote add nbd-exp https://github.com/esignoretti/NBD-experimental.git
git fetch nbd-exp

# 62acc10 is the baseline commit in this repository; the range is the 103 NBD commits.
git cherry-pick 62acc10..nbd-exp/main

# Expect the rename conflict in payload_operation.go, resolve it as described, then:
go build ./... && go test ./internal/adapter/nbd/ ./pkg/adapter/nbd/
```

Two things to check that this repository cannot check for you:

1. **The ten modified files may have moved on.** The merge was verified against `51fc8443`; if `write_payload.go`, `start.go` or `normalize.go` changed after v0.36.0, expect conflicts there. Re-verify each premise — the SDD ledger's own standing hazard is *"audits predate later fixes — re-verify each premise on develop."*
2. **`crash-restart.sh` has run end to end on a real Linux host, not just in a container.** It needs Linux, root and an `nbd-client` built from upstream master (3.26.1 has no `-T`; 3.27.1's `--persist` is silently dead). The run that first passed also found three defects in the script itself — a missing `-t`, a pre-flight that described the file rather than the running server, and no size cross-check — all fixed and described in [`docs/nbd-benchmarks.md`](docs/nbd-benchmarks.md).

## Credit

The NBD adapter was designed and written by **Enrico Signoretti** as a branch of [DittoFS](https://github.com/marmos91/dittofs) by **Marco Moschettini**, and this repository is an extraction of that branch. DittoFS is MIT licensed; see [LICENSE](LICENSE).

The design, the plans, the SDD artifacts, and every measurement in this README are from that work. The code review that found the two P0/P1 bugs and the fixes for them were done during the extraction.

Nothing here is a fork of DittoFS's direction or an official DittoFS release. It is one feature branch, published so it can be reviewed and merged.
