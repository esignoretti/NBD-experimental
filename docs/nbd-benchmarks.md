# Benchmarks and test results

Every number on this page comes from running the adapter, not from estimating it. The
method is described alongside each table so you can judge the figure rather than trust it.

Three results are worth reading even if you skip the tables:

- **The depth plateau reproduces.** Random 4 KiB reads climb steeply to queue depth 4 and
then gain nothing through depth 64. That is the measurement that motivates the pipelining
work, and it is the one number in this document that should change when that work lands.
- **Crash/reconnect now passes end to end.** The rig existed but had never been executed;
  running it found a real bug in the rig itself (fixed in `5480f219`), and after the fix the
  test passes and its negative control fails in exactly the way it must.
- **The live server's numbers do not reproduce the ones previously published**, and the
  reason is the backing store, not the adapter. That investigation is written up in full
  because a lower number with an explanation is worth more than a higher one without.

---

## What was tested

| | |
|---|---|
| Branch | `nbd-on-036` at `5480f219` |
| macOS arm64 | Go 1.27.2, `qemu-img`, `qemu-system-x86_64`, `qemu-nbd` 10.x |
| Linux amd64 | 2 vCPU host, kernel 6.8, `nbd-client` built from upstream `master` (`9c69969`) |
| Storage backends | RAM-backed fixture (isolates the adapter) and the live S3-backed server |
| Code size | 6,288 lines — 2,234 production, 4,054 test; 97 test functions across 9 files |

Two backends are used deliberately. The RAM-backed fixture measures the adapter with the
storage layer removed from the equation; the S3-backed server measures the whole system.
They answer different questions and the tables say which one produced which.

---

## 1. Functionality

| Check | Result |
|---|---|
| `go build ./...` | pass |
| `go vet` on both NBD packages | clean |
| `gofmt -l` | clean |
| `go test -race` on both NBD packages | **97 top-level tests, 120 including subtests — all pass, 0 fail, 0 skip** |
| `go test ./...` across the whole repository | **151 packages ok, 1 failure — pre-existing and unrelated** (see below) |
| Differential oracle against `qemu-nbd` | pass |

### The one repository-wide failure is not from this branch

`go test ./...` reports one consistent failure, plus a second that is intermittent. Neither is
caused by the NBD work, and both were confirmed rather than assumed:

**`test/harness/known`** fails with `declare: -A: invalid option` from
`test/common/known-failures.sh:38`. macOS ships bash 3.2, and associative arrays need bash 4.
This fails on every run, and was verified pre-existing by stashing this branch's changes and
re-running: it fails identically on the unmodified tree.

**`TestBLAKE3FasterThanSHA256`** in `pkg/block` is a load-sensitive benchmark gate
(`ratio=0.47x` against a `>= 0.50x` threshold). It fails on some runs and passes on others —
observed both `D-41 gate FAILED on arm64: BLAKE3 is 0.47x SHA-256` and a clean `ok 5.180s` for
the same tree within minutes of each other. `pkg/block` is not among the files this branch
touches: `git diff --name-only origin/develop...HEAD | grep -c '^pkg/block/'` returns `0`, so
the branch cannot have caused it.

### The oracle

`internal/adapter/nbd/oracle_test.go` drives the real handler against a real independent NBD
client and compares wire behaviour. Measured transmission-flag words:

```
DittoFS  = 0x006d
qemu-nbd = 0x0c6d
ours-only    = 0x0000     (we advertise nothing qemu-nbd does not)
theirs-only  = 0x0c00     (bits deliberately not set)
```

`ours-only = 0x0000` is the claim worth checking: the adapter advertises no capability it
cannot honour.

---

## 2. A real kernel booted from a real export

`test/nbd/boot-qemu.sh` boots an Alpine `linux-virt` 6.6.142 kernel under QEMU, formats and
mounts the NBD device, writes a file, and a second VM reads it back.

```
INIT: mkfs.ext4 /dev/nbd0 (first boot)
[    0.254854] nbd0: detected capacity change from 0 to 2097152
[    0.348866] EXT4-fs (nbd0): recovery complete
[    0.354275] EXT4-fs (nbd0): mounted filesystem cf1072b2-3bd7-4fc3-9ebc-9db50b490140 r/w
INIT: READBACK hello.txt: [written-by-the-kernel-client]
INIT: OK remount — the filesystem and its contents survived a fresh VM
boot-qemu.sh: VERDICT: PASS
```

The second boot runs `--no-mkfs` in a fresh VM: it must find an existing filesystem, remount
it, and read the file the first boot wrote. Both pass against a 1 GiB export.

**A trap worth recording.** The rig ships a prebuilt initramfs that hardcodes `export/owned`
on port 10809 with `do_mkfs=0`. Running it without `--initramfs-dir` therefore boots against
whatever is listening on 10809, not against the export you think you configured — and it
still prints `VERDICT: PASS`. My first run "passed" this way. The tell was in the transcript:
an ext4 UUID from a different day, an 8 MiB capacity, and `--no-mkfs` printed although the
flag was never passed. Always generate a fresh initramfs for the export under test.

---

## 3. Crash and reconnect

This is the strongest functional result here, and it was previously unproven.

`test/nbd/crash-restart.sh` SIGKILLs the server mid-flight, proves the kernel client's I/O
*pauses* rather than failing (that is `-T` reaching the kernel), then restarts the server
inside the window and proves the read *resumes with the pre-crash bytes* (that is `-p`
reconnecting).

```
=== SIGKILL the server FIRST, before issuing the read
  server pid 1897295 killed
=== issue the read — it must PAUSE, not fail (that is -T reaching the kernel)
  read still blocked at t=2s
  read still blocked at t=4s
=== restart the server inside the window (at t=4s, window 30s)
  restart command returned at t=5s
  read still blocked at t=8s ... t=28s
VERDICT: read paused (blocked for >=2s), server restarted at t=5s inside the 30s window,
read completed status=0 elapsed=31s with PRE-CRASH-PATTERN-AAAA.
```

**The negative control fails the way it must.** Run with `--keep-dead` (no restart), the same
rig must report a failure at the window rather than a hang or a success:

```
VERDICT (negative control): the read failed with status 1 after 10s with no server running
— that is -T reaching the kernel.
```

Without this control a "pass" proves nothing: a read served from the page cache returns
instantly and the kill lands on an idle device. The rig's own header documents two earlier
orderings that looked right and proved nothing; both were tried and rejected.

**The version trap is real.** `nbd-client` 3.26.1 from the distribution packages has no `-T`
at all, and the rig refuses it by name with an accurate explanation. Building from upstream
`master` works. The claims in `test/nbd/README.md` about which client versions can reconnect,
and about the client needing the init network namespace, are correct as written.

### A bug in the rig, found by running it

The rig waited for the block *node* before writing its pattern:

```sh
for _ in $(seq 1 100); do [ -b "$device" ] && break; sleep 0.1; done
```

Measured on a real export, **the node appears while `/sys/block/nbdN/size` is still 0**. A
write at that instant fails `ENOSPC` on a perfectly healthy volume, and the script then
blamed the adapter identity — sending the reader after a permission bug that does not exist:

```
dd: error writing '/dev/nbd6': No space left on device
crash-restart.sh: REFUSING: ... the volume and volumes/ must be owned by UID/GID 65534 ...
```

The fix waits for the capacity as well as the node. Committed as `5480f219`. This is the
argument for executing a rig rather than reviewing it: the script was carefully written,
thoroughly commented, and its first real run produced a confidently wrong diagnosis.

---

## 4. Performance: random 4 KiB reads by queue depth

**Method.** `qemu-img bench` over the export, 10,000 operations of 4 KiB at `-t none`,
best of 3, on the 2-vCPU Linux host against a RAM-backed fixture. Best of 3 because the
first run of a series pays for connection setup and cold metadata.

| Depth | 1 | 2 | 4 | 8 | 16 | 32 | 64 |
|---|--:|--:|--:|--:|--:|--:|--:|
| IOPS, sweep A | 7,686 | 11,325 | **14,104** | 14,450 | 13,297 | 14,124 | 13,586 |
| IOPS, sweep B | 9,090 | 13,297 | 16,260 | **15,822** | 15,673 | 15,797 | 14,556 |

**Read the shape, not the peak.** Throughput rises steeply to depth 4, then stops improving:
every depth from 4 to 64 lands in the same band, with no trend either way. Repeated
single-depth runs confirm it — depth 1 measured 10,384 / 8,319 / 9,165 across three
repetitions, depth 8 measured 16,528 / 15,625 / 13,351, depth 32 measured
16,949 / 15,015 / 15,337. Depth 32 does not beat depth 8 by any margin that survives the
noise.

That plateau is the serialization signature: the dispatch loop handles one command at a
time, so once a single command's latency is saturated, more outstanding requests buy
nothing.

### The same plateau, reproduced in a unit benchmark

The sweeps above need a Linux host, a QEMU guest and a fixture. `bench_test.go` reproduces
the shape in-process, against a store that sleeps 100 µs per read — latency in the store is
what makes queue depth matter, because against a nanosecond store the adapter is CPU-bound
and the question is meaningless.

```
BenchmarkConnection_ReadDepth1-8           2000   157072 ns/op   26.08 MB/s
BenchmarkConnection_ReadDepth8-8           2000   137451 ns/op   29.80 MB/s
BenchmarkConnection_ReadDepth32-8          2000   139021 ns/op   29.46 MB/s
BenchmarkConnection_ReadDepth64-8          2000   139474 ns/op   29.37 MB/s
BenchmarkConnection_ReadStoreCeiling-8     2000   120447 ns/op   34.01 MB/s
```

Depth 1 to 8 buys 12%; 8 to 64 buys nothing. The store ceiling — the same latency store
driven directly, no protocol in the way — is 120 µs against the adapter's 137 µs, so **88%
of each request is spent waiting on the store** and the remaining 12% is everything the
protocol does. Queueing requests behind a serialized dispatcher cannot recover that wait.

This is the number the pipelining work is meant to move. A flat depth-N result after it
lands is a regression, not noise. Run it with:

```sh
go test -run '^$' -bench 'BenchmarkConnection_Read' -benchtime 2000x ./internal/adapter/nbd/
```

`CAN_MULTI_CONN` is a separate lever on the same problem, and it is now advertised. Note
what the flag does and does not do: the spec says that *in the absence* of it, "clients
SHOULD NOT multiplex their commands over more than one connection to the export". Presence
removes that prohibition — it does not make a client open a second connection. A client that
chooses to (QEMU with multiple `-e` blocks, `nbd-client -C N`, or an application opening two
sessions) gets a separate dispatcher per connection, so the ceiling above applies per
connection rather than per export. It does not raise the ceiling *within* one connection —
that is what pipelining is for.

**Two honesty notes about these numbers.**

First, they do not match the figures previously published for this branch (5,328 / 8,880 /
8,955 / 9,245 / 8,918 at depths 1/8/16/32/64). The *shape* agrees — depth 1 is far below the
rest, and past depth 8 nothing improves. The absolute values differ by 30–60%, and the run-to-run
variance on this host is itself ±15–20%. Both sets of numbers were produced by the same
command against the same fixture; the host state differed. Treat the plateau as the finding
and any single absolute figure as approximate.

Second, an intermediate set of runs measured much lower (roughly 5,000–9,000 IOPS across
all depths) because the fixture's journal had grown to 1.4 GB on a partition that was
90–96% full. Those runs measured disk pressure, not the adapter. The tables above come from a
fresh journal at 79% disk use, verified before and after each sweep.

### Sequential

Same fixture, 500 × 1 MiB, best of 3:

| | depth 1 | depth 8 |
|---|--:|--:|
| Sequential read | 1,259 MiB/s | 1,302 MiB/s |

Sequential reads are **flat in depth** as well — the same bottleneck, seen from the
throughput side rather than the IOPS side.

**Sequential writes could not be measured reliably on this host** and no number is claimed.
The fixture journals every write to disk, and 800 × 1 MiB runs grew the journal from 875 MB
to 1.4 GB on an 8 GB root partition, driving it from 90% to 96% full and inflating elapsed
time as the run progressed. Shorter runs (100 × 1 MiB) measured 0.20–0.92 s with the spread
coming from filesystem state rather than the adapter. A write benchmark here needs a
separate, larger backing device; measuring it on the system disk would report the disk.

These are RAM-backed read figures and say nothing about object-storage bandwidth. They bound
what the adapter itself can do when the storage layer is not the constraint.

---

## 5. The live S3-backed server: why its numbers differ

The same depth sweep against the live server (`62.210.237.51`, S3-backed) gives:

| Depth | 1 | 8 | 16 | 32 |
|---|--:|--:|--:|--:|
| IOPS | 92 | 94 | 95 | 110 |

Roughly 1/50th of the fixture, and flat. This is not a regression in the adapter, and the
investigation is worth recording because the first several hypotheses were wrong.

**Ruled out:**

- *Host contention.* `load average: 0.66, 1.12, 1.38`; the server process at ~1% CPU with 12
  threads; `nbd0` inflight `0 0`.
- *Network round-trips to object storage.* Interface counters are **unchanged** across a
  benchmark run, so reads are not leaving the machine.
- *The benchmark harness itself.* The identical command against a local file reaches
  100,000 IOPS, so the tool is not the limit.

**What it actually is:** the local tier is not retaining reads.

- A single cold 4 KiB read costs **13.75 ms**.
- The block store reports `hit-ratio: 0.40` with a cache life expectancy 50th percentile of
  **2.00 s** — entries are evicted almost immediately.
- 30 reads of the *same* offset take 2.3 s; 30 reads at *distinct* offsets across 1 GiB take
  3.7 s. The gap is small, which rules out per-request connection overhead as the dominant
  cost and points at a per-read tier miss.

So every 4 KiB read pays a miss. That is a **block-store and cache-configuration finding, not
an NBD-adapter finding** — and it is worth chasing on its own, because 92 IOPS at depth 32 is
not a usable demo regardless of protocol. It also means the previously published live numbers
came from a server in a different cache state than today's.

---

## 6. Measurement traps that produce wrong numbers

Recorded because each one produced a plausible-looking figure that was wrong.

**A loaded machine.** An early local sweep on a busy Mac read 2,751 IOPS at depth 1 where the
same machine idle reads 5,157 — a 48% error, caused by an unrelated process at 100% CPU and
load average 6–12. Check `load average` before trusting any throughput number, and prefer a
quiet machine over a convenient one. The same caveat applies to the host in §4: its
depth-1 figure varied from 7,686 to 10,384 across sweeps with nothing changed but timing.

**`qemu-img bench -S 0`.** The default step equals the block size, which re-reads one offset
forever: that measures the cache, not the export. Distinct offsets need an explicit `-S`.

**A "pass" from a stale initramfs.** Covered in §2 — the rig can report `VERDICT: PASS`
against an export you did not configure. Read the transcript, not just the verdict.

**A fixture that fills its own disk.** Covered in §4 — the RAM-backed fixture journals writes
to the same small partition it lives on. A write sweep can drive that partition to 96% full
and then the numbers report filesystem pressure. Check `df` before and after, and reset the
journal between sweeps.

**A memory-backed remote store.** The crash-restart rig asserts that bytes survive a server
restart. Against a `type: memory` remote store with no durability override, every restart
begins with an empty tier and the assertion measures the harness, not the server. The run
must use a store that is actually crash-durable, or the test is meaningless.

---

## 7. Reproducing

The functional suite needs nothing beyond a Go toolchain:

```sh
go build ./...
go vet ./internal/adapter/nbd/ ./pkg/adapter/nbd/
go test -race -count=1 ./internal/adapter/nbd/ ./pkg/adapter/nbd/
go test -count=1 -v -run Oracle ./internal/adapter/nbd/
```

The kernel boot needs `qemu-system-x86_64` and the rig's fetched artifacts, and **must** be
given a freshly generated initramfs for the export under test:

```sh
./test/nbd/fetch-artifacts.sh
./test/nbd/boot-qemu.sh --initramfs-dir /tmp/initramfs-x \
    --volume-path /path/to/share --port 10900 --mkfs
./test/nbd/boot-qemu.sh --initramfs-dir /tmp/initramfs-x \
    --volume-path /path/to/share --port 10900 --no-mkfs
```

The crash test needs Linux, root, a block device, and an `nbd-client` built from upstream
`master` — no released version can reconnect:

```sh
sudo NBD_CLIENT=/path/to/master/nbd-client ./test/nbd/crash-restart.sh \
    --port 10900 --export export/boot --device /dev/nbd6 \
    --dead-timeout 30 --restart-cmd 'systemctl restart dittofs' \
    --volume-path /path/to/share

# and the negative control, which must FAIL at the window:
sudo NBD_CLIENT=/path/to/master/nbd-client ./test/nbd/crash-restart.sh \
    --port 10900 --export export/boot --device /dev/nbd6 \
    --dead-timeout 10 --keep-dead --volume-path /path/to/share
```

The depth sweep:

```sh
for d in 1 2 4 8 16 32 64; do
  qemu-img bench -d $d -c 10000 -s 4k -t none nbd://127.0.0.1:10900/export/boot
done
```

---

## What these results do and do not show

**Do show:** the protocol works end to end against real clients and a real kernel; the adapter
advertises nothing it cannot honour; crash/reconnect genuinely pauses and resumes I/O; the
adapter is coherent across connections, so `CAN_MULTI_CONN` is safe to advertise; and the
dispatch loop is serialized, with the depth plateau as direct evidence — reproduced both by an
external sweep and by an in-process benchmark against a latency-injecting store.

**Do not show:** production readiness. There is no pipelining, no CI coverage of these rigs,
and the live server's cache behaviour needs its own investigation before any number from it
means anything. The performance figures above describe a prototype with a known bottleneck
that is scheduled to be removed — not a system that has been tuned.

**Two capabilities are absent by decision, not by omission,** and both are documented rather
than left to be discovered:

- `BLOCK_STATUS` and structured replies — so `qemu-img map` and `nbdinfo --map` cannot be
  used, and `qemu-img convert` copies every byte instead of only the allocated ones. The gate,
  its cost and the checklist to lift it are in [`nbd-block-status.md`](nbd-block-status.md).
- Snapshots — an export is a live view of one volume, so there is no point-in-time copy to
  read from. Guest-side file-level backup (`tar`, `rsync`, `restic`, `borg`) is unaffected,
  because it never asks the block layer.
