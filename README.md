# NBD-experimental

An NBD (Network Block Device) adapter for [DittoFS](https://github.com/marmos91/dittofs), extracted for review and merge proposal.

**Status:** feature-complete and working, with two known bugs fixed and two open work items. Never merged upstream. See [Status](#status).

**This is not a standalone project.** It is an extraction of one feature branch — 77 commits — from the DittoFS repository. Every file sits at its exact upstream path so that the commits apply 1:1 to DittoFS. See [Merging upstream](#merging-upstream).

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
| Payload-length enforcement (advertised 32 MiB maximum) | ✅ |
| `BLOCK_STATUS`, structured replies, meta-context, `STARTTLS` | ❌ out of scope by design |

`BLOCK_STATUS` and structured replies are deliberately absent, not unfinished: DittoFS's block store cannot answer "is this range allocated?" truthfully without leaking residency information across shares. The spec's gate for this is in `docs/superpowers/specs/2026-10-08-dittofs-nbd-block-protocol-design.md` §7. `STARTTLS` is refused rather than ignored — a client that believed its session was encrypted and was not has no way to notice.

### What was proven, and how

This is the part worth reading before trusting any of the above.

**A real kernel booted from a real NBD export.** Not a mock, not a loopback socket: `test/nbd/boot-qemu.sh` boots an Alpine `linux-virt` 6.6.142 kernel under QEMU (hvf on macOS, kvm on Linux), `mkfs.ext4` + mounts the NBD device, writes `hello.txt`, and a *second* VM reads it back. The recorded UUID `853580a8-d9e6-4789-ace8-94180888d0e2` in the rig's README is from that run.

**A differential oracle against `qemu-nbd`.** `test/nbd/run-oracle.sh` and `internal/adapter/nbd/oracle_test.go` drive the real handler with a real independent NBD client and compare wire behaviour. Measured transmission-flag words:

```
DittoFS  = 0x006d
qemu-nbd = 0x0c6d
ours-only    = 0x0000     (we advertise nothing qemu-nbd does not)
theirs-only  = 0x0c00     (bits we deliberately do not set)
```

**A crash/reconnect mechanism proven by hand.** `test/nbd/crash-restart.sh` is written but has never run end to end — it needs Linux, root, and an `nbd-client` built from upstream `master`. The mechanism itself was proven in a container: with `-persist -T 60`, a read blocked at t=4s/8s/12s while the server was down and completed with `exit=0 elapsed=14s` when it returned. Getting there required finding that `nbd-client` 3.26.1 has no `-T` at all and 3.27.1's persist is silently dead on the netlink path (`persist_mode` is never assigned `true`) — only upstream `master` works.

### Measured performance

Random 4 KiB read IOPS by queue depth, against a live 2-vCPU host:

| Depth | 1 | 8 | 16 | 32 | 64 |
|---|---|---|---|---|---|
| IOPS | 5,328 | 8,880 | 8,955 | 9,245 | 8,918 |

**Flat from depth 8.** That plateau is the finding that motivates the open work below: the dispatch loop handles one command at a time, so adding queue depth buys nothing past the point where a single command's latency is saturated.

Two other numbers worth knowing:

- **Chunk size was the single biggest win.** The default block-store profile chunks at 1 MiB, so a 4 KiB random read that misses cache fetches 1 MiB — up to 256× amplification. Setting `chunk_size: 131072` for VM images took depth-1 IOPS from 847 to 5,328 (**6.3×**). Reads never re-chunk, so this only affects newly written data.
- **Sequential** read 46–52 MiB/s (the device does 80), write 169 MiB/s, fsync-per-4KiB-write 208 IOPS at 4.15 ms p99.

---

## Status

### Fixed in this repository

Two bugs found in a full code review and fixed on top of the branch. Both have regression tests that were verified to fail without the fix.

**P0 — a WRITE arriving during drain desynchronized the stream.** The draining branch answered `NBD_ESHUTDOWN` without consuming the payload the client had already sent, so the next header was parsed out of payload bytes. The connection then died on a later command with `NBD_EBADMAGIC` — the exact opposite of the branch's purpose, which is to pause the client for reconnect rather than error the device. Commit `89f2e4a`.

**P1 — the handshake was unbounded.** `ServeHandshake` armed no deadline, the base adapter sets none on an accepted socket, and the dispatch loop's first arm only runs after the handshake returns. A client could connect, send nothing, and hold a goroutine and a file descriptor forever — before naming an export, so before any authorization decision exists to fail. Fixed with a `Timeouts.Read` deadline plus a `ctx` poll in the option loop, so a shutdown also aborts a handshake in progress. Commit `69e80b8`.

### Open work

**1. Pipelining — planned, not started.** This is the fix for the depth plateau. The design is in `docs/superpowers/plans/2026-10-19-nbd-pipelining.md`: a reader goroutine, a worker pool, a single ordered writer; `MaxInFlight` default 4 / max 64; a `maxInFlightBytes` semaphore; FLUSH as a barrier via `sync.Cond` on a pending counter; handlers changed to return a `reply{cookie, nbdErr, payload}` value instead of writing to the socket themselves. The falsifiable prediction: *depth 32 must exceed depth 8 by a wide margin after the change.*

Note that the FLUSH barrier's correctness currently rests on serialization — `handlers_control.go` says so explicitly. Pipelining invalidates that argument and the barrier needs real in-flight tracking.

**2. `crash-restart.sh` needs a Linux host** with KVM, root, and an `nbd-client` built from `master`. See above.

**3. Missing artifacts.** `bench-fio.sh` is cited by the pipelining plan and does not exist. "The deploy script" is referenced twice in `test/nbd/deploy/config-server2.yaml` and does not exist.

**4. No CI coverage.** `nbd` appears in no GitHub workflow and is not in `test/required-tests.json`. Everything above is green locally and enforced nowhere.

### Known minor issues

Recorded, not fixed. The full list of 29 deferred minors is in `docs/sdd/progress.md`.

| Issue | Where |
|---|---|
| `max_message_size`, `default_export`, `uid`, `gid`, `timeouts.*` have mapstructure tags but `createNBDAdapter` reads only four keys — the first and second have *zero* consumers anywhere | `pkg/adapter/nbd/config.go` |
| UID/GID `0` is unexpressible (0 doubles as "unset"), so "run as root" cannot be configured | `config.go:121` |
| `PunchHole` is the only block-store call without a request deadline | `handlers_write.go:265` |
| `handleFlush` does not validate flags, unlike every other command | `handlers_control.go:22` |
| No buffer pooling on READ/WRITE — `make` per command, up to 32 MiB | `handlers_read.go:88` |
| 11 dead protocol constants and 4 test-only codec functions | `protocol.go` |
| `handleRead` ignores the returned `n` (safe today: the journal always fills the buffer on nil error) | `handlers_read.go:96` |
| No read-permission check on the READ path (write path is checked) | `handlers_read.go` |
| Export size is a handshake-time snapshot, so a truncate over NFS/SMB is invisible to a live NBD connection | `volume.go:146` |
| `validate()` only checks the port | `config.go:137` |

---

## Health

At the time of extraction, in the parent repository:

```
go build ./...                                          OK
go vet ./internal/adapter/nbd/... ./pkg/adapter/nbd/...  clean
gofmt -l                                                 clean
go test -race -count=1 (both packages)                   ok 13.4s / ok 3.2s
go test -run Oracle                                      both PASS
```

**97 test functions** across 9 test files: 3,758 lines of tests against 1,880 lines of production code in `internal/adapter/nbd`, plus 296 against 354 in `pkg/adapter/nbd`. The rig under `test/nbd/` is another 1,973 lines of shell.

---

## Layout

Every path below is where the file lives in DittoFS, unchanged.

```
internal/adapter/nbd/           5,638 LOC — the wire layer (1,880 prod + 3,758 test)
  protocol.go                     wire codec: magics, header encode/decode
  handshake.go                    greeting, option haggling, export resolution
  connection.go                   per-connection dispatch loop, Handler, BlockStore
  handlers_read.go                NBD_CMD_READ
  handlers_write.go               NBD_CMD_WRITE, TRIM, WRITE_ZEROES, payload drain
  handlers_control.go             NBD_CMD_FLUSH, the durability barrier
  volume.go                       RuntimeExporter: <share>/<volume> → Export
  payload_size.go                 the advertised-maximum bound
  reply.go                        reply writers

pkg/adapter/nbd/                  650 LOC — the framework side (354 prod + 296 test)
  adapter.go                      BaseAdapter lifecycle, Serve, NewConnection
  config.go                       Config, defaults, validation
  healthcheck.go                  degraded-export reporting

test/nbd/                        the proof rigs
  README.md                       32 KB: topology, every command, every finding
  boot-qemu.sh                    40 KB: boots a real kernel from an export
  crash-restart.sh                36 KB: server-down/reconnect (needs Linux+root)
  fetch-artifacts.sh              20 KB: downloads the kernel + nbd.ko
  run-oracle.sh                   differential run against qemu-nbd
  deploy/config-server2.yaml      demo host config, with the memory budget

docs/superpowers/specs/          the design spec (§1–§13)
docs/superpowers/plans/          the 8-task implementation plan, and the pipelining plan
docs/sdd/                        the SDD record: briefs, reports, review diffs, progress ledger
```

### The ten files outside the adapter

The adapter is not entirely self-contained. These existing DittoFS files were modified, and they are why a merge needs care:

| File | Change |
|---|---|
| `cmd/dfs/commands/start.go` | `createNBDAdapter` + `case "nbd"` registration |
| `pkg/controlplane/models/adapter.go` | `"nbd"` accepted as an adapter type |
| `internal/adapter/common/write_payload.go` | FLUSH barrier: commit the metadata half, not just the block store |
| `internal/adapter/common/normalize.go` | `ErrAccessDenied` → `EPERM` mapping |
| `internal/adapter/common/read_payload.go`, `payload_operation.go`, `doc.go` | shared helpers the NBD handlers reuse |
| `internal/adapter/nfs/v4/handlers/content_errmap_wiring_test.go` | test adjusted for the shared helper |
| `.gitignore` | `test/nbd/artifacts/` (56 MB of downloaded kernel + modules) |

The `write_payload.go` change is a genuine bug fix that affects NFS and SMB too: the FLUSH barrier committed only the block-store half, leaving a client told "durable" with metadata a restart could lose.

---

## Merging upstream

The point of this repository is to make a merge proposal possible, so the history was reconstructed to make one clean.

**All 39 files are byte-identical to the original branch.** Verified at extraction, per file:

```
$ git ls-tree -r 498a9d9   # the last commit before the review fixes
$ ...compare each blob against the same path on nbd-on-036
mismatches: 0 of 39
```

(Whole-tree hashes necessarily differ — the original branch has 3,305 files and this extraction has the 39 the NBD work touches. What matters for a patch is that every one of those 39 blobs is identical.)

**The history is 77 NBD commits** — the same commits, same authors, same timestamps, same messages, in the same order — replayed onto the DittoFS commit they were originally based on (`62acc10`). Unrelated upstream commits that the extraction would otherwise have carried are dropped; nothing about the NBD work is lost.

**What changed from the original branch:**

- `merge: NBD adapter onto v0.36.0` is excluded — it merged upstream into the branch, which is not meaningful in an extraction.
- Five commits on top: the two fixes above (`89f2e4a`, `69e80b8`), a redaction commit (`e5b8bf2`), the SDD record, and this README.
- `test/nbd/deploy/config-server2.yaml` had the real deployment's JWT secret and public address; both are redacted.
- `docs/sdd/` is new here — the SDD artifacts lived in `.superpowers/`, which is gitignored upstream.

### To propose the merge

```bash
# In a DittoFS clone at the commit this was based on:
git remote add nbd-exp https://github.com/esignoretti/NBD-experimental.git
git fetch nbd-exp

# Option A: cherry-pick the range (keeps messages and authors)
git cherry-pick 62acc10..nbd-exp/main

# Option B: merge (preserves the 77-commit history as one merge)
git merge --no-ff nbd-exp/main
```

Then re-run the gate. Two things to check that this repository cannot check for you:

1. **The ten modified files may have moved on.** The branch was based on `62acc10`; if `write_payload.go` or `start.go` changed since, expect conflicts there. Re-verify each premise — the SDD ledger's own standing hazard is *"audits predate later fixes — re-verify each premise on develop."*
2. **`crash-restart.sh` still has not run end to end**, and there is still no CI job. Both are worth doing before a merge rather than after.

---

## Credit

The NBD adapter was designed and written by **Enrico Signoretti** as a branch of [DittoFS](https://github.com/marmos91/dittofs) by **Marco Moschettini**, and this repository is an extraction of that branch. DittoFS is MIT licensed; see [LICENSE](LICENSE).

The design, the plans, the SDD artifacts, and every measurement in this README are from that work. The code review that found the two P0/P1 bugs and the fixes for them were done during the extraction.

Nothing here is a fork of DittoFS's direction or an official DittoFS release. It is one feature branch, published so it can be reviewed and merged.
