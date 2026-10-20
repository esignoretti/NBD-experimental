## Task 8: KVM boot harness and crash rig

**Files:**
- Create: `test/nbd/boot-qemu.sh`, `test/nbd/crash-restart.sh`
- Modify: `test/nbd/README.md`

**Interfaces:**
- Consumes: a DittoFS server with the NBD adapter, `qemu-system-*`, a kernel and initramfs, root.
- Produces: two scripts whose exit status is the verdict.

**This task cannot run its crash half on the development machine** (macOS arm64: the crash rig needs a Linux guest with a `master`-built `nbd-client`, root, and netlink). **The boot half can run here** — HVF is a working accelerator on this host (`qemu-system-aarch64 -accel help` lists `hvf` and `tcg`; `-accel kvm` is Linux-only and fails with `invalid accelerator kvm`), so Step 1 is testable and the implementer should test it rather than reasoning about it. The deliverable is the harness plus a recorded run on a Linux host for Step 2. The scripts must fail loudly and specifically when their prerequisites are absent rather than silently skipping.

- [ ] **Step 1: Write `boot-qemu.sh`**

Topology, exactly as the spec's §9.5:

```
host:                   -kernel Image -initrd initramfs.cpio.gz   <- no disk, no NBD
initramfs:              modprobe nbd
                        nbd-client -N export/boot 10.0.2.2 10809 /dev/nbd0 &
                        mkfs.ext4 /dev/nbd0        <- FIRST BOOT ONLY
                        mount /dev/nbd0 /root; switch_root
NBD (DittoFS):          the root filesystem
```

The `&` is load-bearing: with `-p` the client never returns (see Step 2), so a foreground
`nbd-client` would hang before `switch_root` and the boot would never reach the volume.

**Two spellings in that line are load-bearing, and both fail without naming the cause:**

- **`-N <export>` is mandatory.** Omitting it does not mean "no export name" — `nbd-client` sends an **empty** one (`master`: `if(!cur_client->name) cur_client->name = "";`, `nbd-client.c:1504-1505`, with only a warning about oldstyle). DittoFS refuses it, because `RuntimeExporter.Resolve` requires a `/` with a non-empty share on the left and a non-empty volume on the right (`sep <= 0 || sep == len(name)-1` → `ErrUnknownExport`). Measured: resolving `""` returns `nbd: unknown export`. The client reports a handshake failure that never mentions the missing flag. The name must be the same one the oracle uses — `<share>/<volume>` with no leading slash, e.g. `export/boot`.
- **Mount `/dev/nbd0`, not `/dev/nbd0p1`.** The export is a single file served whole and nothing in this path writes a partition table; `mkfs.ext4` goes directly on the device. A `p1` suffix assumes a partition table no step creates, so the mount fails with `no such device` after both the handshake and the `mkfs` have already succeeded.

- **No data disk.** The root filesystem arrives over NBD. A `-drive` holding the volume would test the wrong path.
- **No `hostfwd` rule.** `hostfwd` forwards a *host* port *into the guest*, which is the opposite of what this topology needs and would also try to bind host port 10809 where DittoFS is already listening. The guest initiates the connection outward to `10.0.2.2`, QEMU's user-net gateway to the host, and a guest-initiated outbound connection crosses SLIRP with no rule — the same path the guest uses to reach the outside world.
- **No UEFI-boot-from-NBD.** EDK2 has no NBD driver, and `-M virt` on arm64 has no BIOS to run EDK2 with in the first place. The boot path is `-kernel`/`-initrd` handed over directly, then root-on-NBD via `switch_root`.
- The script must build the initramfs with the `nbd` module and `nbd-client`, or refuse with a message naming what is missing.
- **The recovery environment must not be on NBD.** The script must not run `fsck` from inside the guest: the guest's root is the volume under test, so an `fsck` there is judging a crash it is also recovering from. The check belongs on the host, against the volume read as a file over NFS — which is the B1 property (spec §2.2) earning its place.
- Record which cache is in the path, in the script's own output: in this topology the guest's kernel NBD client speaks TCP directly to DittoFS, so there is no QEMU block layer, and a `-drive`'s `cache=` mode is irrelevant. Print the guest page-cache state and the share's `CommitAck` value so the run is self-describing.

Reference launch (from the spec):

```
qemu-system-aarch64 -M virt -accel kvm -cpu host -m 2G \
  -kernel Image -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/init" \
  -netdev user,id=n0 -device virtio-net,netdev=n0 \
  -display none -serial stdio
```

**`-accel kvm` is Linux-only. On macOS the accelerator is `hvf`, and it works on this host** — measured: `qemu-system-aarch64 -accel help` on this machine lists `hvf` and `tcg` only, and `-accel kvm` fails with `invalid accelerator kvm`. The script must **select** the accelerator rather than hardcode one:

```sh
accel=hvf                      # macOS
[ "$(uname -s)" = Linux ] && accel=kvm
```

This is not cosmetic. It changes what the harness can prove and where: **HVF on an Apple-silicon host is a working KVM substitute for the boot test**, so the boot half of Task 8 is runnable here, while the crash half still needs a Linux guest with a `master`-built `nbd-client` (Step 2). The plan's earlier claim that *"this task cannot run on the development machine"* is therefore too strong for Step 1 and should not be repeated to the implementer as a reason not to try.

**Three things about that line were wrong in the first draft, and each one fails in a way that names neither the cause nor the fix.**

- **`-cpu host` requires an explicit `-accel`.** Measured on QEMU 11.1.2: `qemu-system-aarch64 -M virt -cpu host` alone → `qemu-system-aarch64: Invalid CPU model: host`, followed by the list of valid models (`cortex-a7 … neoverse-n2, max`). Without an accelerator QEMU defaults to TCG, and `host` is meaningless there because TCG has no host CPU to mirror. `-accel kvm -cpu host` and `-accel hvf -cpu host` are both accepted; `-cpu max` is the portable substitute. **And the accelerator must stay explicit rather than relying on `fallback=allow`:** measured, `-accel kvm,fallback=allow` on a machine without KVM prints `invalid accelerator kvm` — it does not silently degrade — so the explicit form is safe and the script should pass it. A silent TCG fallback would be much worse than a refusal here, because TCG is slow enough to make the dead-connection window measurements meaningless while still passing.
- **There is no `boot.img`.** The first draft's `-drive if=virtio,file=boot.img,format=raw` implies something creates a bootable disk, and nothing in this plan does: `grep` for `boot.img` finds only the launch line and its own description. On `-M virt` an arm64 guest has no BIOS, and the plan never says how a kernel would be found on a disk. **Use `-kernel`/`-initrd` instead** — it removes a provisioning step and one more thing that can fail for an unrelated reason. `-kernel` is verified available on the QEMU in use (`qemu-system-aarch64 -M virt -kernel /nonexistent` → `could not load kernel '/nonexistent'`, i.e. the option is parsed and acted on).
- **`-display none -serial stdio` is required** for the guest transcript to reach the host, which is the only evidence the script produces. Without it the boot is silent and a successful boot is indistinguishable from a hung one.

**The volume must be provisioned before the guest boots, and nothing in the plan said so.** `RuntimeExporter.Resolve` looks up the share root, then `volumes/`, then the volume name — and **no command creates a volume**: `grep -rn "volume" cmd/dfsctl/` returns nothing. `internal/adapter/nbd/volume.go:21` records the intent (*"the operator, not the protocol, decides who owns the volumes"*), so provisioning is by hand, through the share's own NFS/SMB mount, and it is the operator's job in production too. The harness must therefore do it explicitly, and the plan's Step 1 must say how, because a script that skips it fails at the handshake with `nbd: unknown export` — the same message an omitted `-N` produces, which is why the two must be distinguished in the script's own output. Minimum sequence:

```
# host, before the guest boots: a share, a local tier, and a volume file of the right size
dfsctl store metadata add ... ; dfsctl store block add ... ; dfsctl share create /export
dfsctl adapter enable nbd
# create volumes/boot THROUGH the share (NFS or SMB), as the configured identity, then:
#   the file's size IS the export size (spec §4.4), and CreateFile zeroes it
#   (pkg/metadata/validation.go:492 ApplyCreateDefaults sets Size = 0), so the size
#   must be set afterwards with SetFileAttributes(SetAttrs{Size: &n})
```

That last note is the trap from Task 5: `CreateFile` with `FileAttr{Size: N}` does **not** produce a file of size N.

**AND THE VOLUME MUST BE CREATED BY THE IDENTITY THE NBD ADAPTER RUNS AS — measured end to end against a live server with a Linux kernel client, and the defect that took longest to find because every intermediate symptom named something else.** The adapter runs as **65534** (`DefaultUID`/`DefaultGID`, `pkg/adapter/nbd/config.go:26-27`), and `applyDefaults` **rewrites a configured `UID == 0` back to 65534** (`:125-129`), so the identity is not configurable away — there is no `--uid` flag on `dfsctl adapter edit` and no control-plane surface for it. A volume created through the share as **any other identity** (the `admin` account's SMB session maps to uid 0) is owned by that identity at mode `600644`; the write then runs as 65534, is not the owner and is not root, so `calculatePermissions` (`pkg/metadata/auth_permissions.go:183`) takes the **other** bits — `r--` — and refuses. Measured at the refusal point with a temporary probe in `handleWrite`: `uid=0 gid=0 mode_oct=600644 acl=nil nbd_uid=65534 nbd_gid=65534`, then `AccessDenied: write permission denied`.

- **`chmod` through the SMB mount does NOT change the mode.** Measured twice, before and after `chmod 666`: `mode_oct=600644` both times, write still refused. A harness that "fixes" this with a `chmod` will appear to have done something and will not have.
- **The `volumes/` directory carries the same requirement and fails earlier:** owned by root at 0755 it refuses the adapter's own file creation with `Permission denied` from the mount, so it must be created by the adapter identity too — or removed and recreated, which is the only reliable repair once it exists.
- `dfsctl user create --uid 65534` is the supported way to name that identity. The share grant must be read-write: a `none` default permission refuses the **resolution** itself, reported as `nbd: unknown export`.
- **Two unrelated causes, one symptom.** The same "write never lands" also appears as `manifest reconcile for <payload> at offset 0 failed: CAS object missing ... chunk not found` when the volume was written by a server instance whose in-memory remote tier has since restarted. So the script must print the export's size and the adapter identity **before** it starts, and must never treat a single failure message as diagnostic on its own.
- **Verified working when both are created as the adapter identity:** the probe reports `uid=65534 gid=65534`, no error follows, and the bytes written through NBD read back byte-exact **through NBD and through the SMB mount independently**.

- [ ] **Step 2: Write `crash-restart.sh`**

```
guest: write + fsync, keep writing
host:  SIGKILL the dfs process mid-flight
host:  restart dfs, same port, same export
guest: I/O resumes without a filesystem error; fsck clean (on the host)
```

Two harness requirements the spec calls non-negotiable, both of which the script must enforce rather than document:

1. **A clean `umount` is not evidence.** The script must SIGKILL the server; it must refuse to pass if the guest unmounted cleanly, because that never exercises the barrier.
2. **Restart inside the dead-connection window.** The kernel's `dead_conn_timeout` (commit `560bc4b399`, `NBD_ATTR_DEAD_CONN_TIMEOUT`) pauses in-flight requests rather than failing them — but a window over ~120 s trips the hung-task detector, so the script must measure and print the actual restart time and fail if it exceeds the configured window.

**THE MECHANISM, CORRECTED — `-T` alone does NOT reconnect, and this is the error the first draft of this step made.** Read from `drivers/block/nbd.c` and `nbd-client.c` on `master`:

```
wait_for_reconnect()                      nbd.c:1101-1114
    if (!config->dead_conn_timeout) return 0;      <- no window => fail IMMEDIATELY
    wait_event_timeout(config->conn_wait,
        DISCONNECTED || live_connections > 0,
        dead_conn_timeout)                          <- waits for USERSPACE to act

grep -c nbd_reconnect_socket drivers/block/nbd.c   => 0   <- kernel never reconnects
```

`live_connections` is incremented **only** by userspace netlink operations (`nbd_add_socket` `:1339`, `nbd_genl_reconfigure` `:1401`). The kernel notices the dead link in `nbd_mark_nsock_dead` (`:307-330`), multicasts `NBD_CMD_LINK_DEAD`, and then **waits for a client to hand it a new socket**. So the two settings do different jobs and the harness needs **both**:

| Setting | What it does | What it does *not* do |
|---|---|---|
| `-T <seconds>` | sets the kernel's patience before failing in-flight I/O | reconnect anything |
| `-p` / `--persist` | runs the client's monitor: `NBD_CMD_LINK_DEAD` → `reconnect_with_backoff` → `netlink_reconfigure` → `NBD_CMD_RECONFIGURE` | set the window (it supplies a 30 s default, which `-T` overrides) |

**`-p` alone gives a 30 s window and reconnects. `-T` alone gives a window and then `BLK_STS_IOERR`.** The script must pass both, and must not describe `-p` as "not this mechanism" — it is the *only* thing that performs the reconnect.

**And `-p` does not daemonize**, which the initramfs script must account for: the persist branch returns before `daemon(0,0)` (`nbd-client.c:1571` returns `persist_mode_main(...)`, `:1580` is the daemon call). Run in the foreground it never returns and `switch_root` is never reached; it must be backgrounded. (Backgrounding is safe across `switch_root`: busybox's `delete_contents` unlinks the initramfs files to free RAM and does **not** kill processes.)

**How to configure it, exactly, and the trap in the obvious spelling.**

- The flag is `-T <seconds>` / `--dead-timeout <seconds>` (`args.c:76`, handled at `:199`). It is **not documented in `nbd-client`'s man page** — `grep -i dead man/nbd-client.8.sgml.in` finds only "deadlock" — so a reader will not discover it from `man nbd-client`. Name the flag in the script's comment and in the README so the harness is self-explanatory.
- It reaches the kernel **only through netlink**, as `NBD_ATTR_DEAD_CONN_TIMEOUT` (`nbd-client.c:185-186`). On the ioctl path the option is parsed, stored, and then **never read**: the only other references to `dead_conn_timeout` in `nbd-client.c` are inside the netlink send. With `-nonetlink`/`-L`, or on a kernel built without netlink support, the script would print a configured window that the kernel never received — the same class of silent failure the whole design exists to eliminate, one layer down.
- **The script must therefore assert the mechanism took effect, not just that it passed the flag.** `-nonetlink`/`-L` is not to be used, and the script must fail if netlink is unavailable rather than falling back. This is checkable before the crash test even starts: `nbd-client` refuses at `nbd-client.c:394 check_conn` when the netlink family will not resolve.
- **`-persist` alone already gives 30 seconds** (`nbd-client.c:188`: `else if (cur_client->persist) NLA_PUT_U64(msg, NBD_ATTR_DEAD_CONN_TIMEOUT, 30); /* Default 30 seconds for persist mode */`). So the default is already inside the hung-task window. The script should still set `-T` explicitly and print the value, because a default that happens to be safe is not a configured one — but it must not *depend* on `-persist` being what supplied the window.
- The kernel side landed in 2017 (`560bc4b39952ed77cdb0000992e9415b0ee89edb`, "nbd: handle dead connections"), so any kernel in support today has it; the requirement to check is `nbd-client`'s netlink support, not the kernel's.

**THE VERSION TRAP, and it is the one that will actually bite.** Everything above is true of `master` and of the **3.27.1** tag. It is **not** true of **3.26.1**, which is what most hosts ship: Debian bookworm has `1:3.26.1-6.1`, Ubuntu 24.04 has `1:3.26.1-2`, Debian trixie/sid has `1:3.27.1-4`. Measured by fetching each tag's `nbd-client.c` and `args.c`:

| | 3.26.1 | 3.27.1 | master |
|---|---|---|---|
| `-T`/`--dead-timeout` | **absent entirely** | present (`args.c:76`, `:198`) | present |
| `NBD_ATTR_DEAD_CONN_TIMEOUT` sent | **never** | only behind the gate below | yes |
| persist over netlink | **no — `-p` is silently ignored** | **no — same bug** | yes |

On 3.26.1 there is no way to set `dead_conn_timeout` at all: the flag does not exist, and its `netlink_configure` sends only `INDEX`, `SIZE_BYTES`, `BLOCK_SIZE_BYTES`, `SERVER_FLAGS` and `TIMEOUT` (`NBD_ATTR_TIMEOUT`, which is `NBD_SET_TIMEOUT` — a different mechanism, see above). Worse, 3.26.1's netlink path does `netlink_configure(...); return 0;` and **never reaches the ioctl reconnect loop**, so `-p`/`--persist` is parsed, stored, and then silently ignored whenever netlink is available (which is the default).

**3.27.1 is not usable either, for a different and more confusing reason.** Its `nbd-client.c:1548` gates the persist monitoring loop on `cur_client->persist_mode`, but the CLI sets `cur_client->persist` (`args.c:186-188`, `client->persist = true`), and **nothing in the tree ever assigns `persist_mode = true`** — verified exhaustively across `nbd-client.c`, `args.c`, `args.h`, `nbdclt.h`, `nbd.h`, `cliserv.h`. The only two writes to it are `= false`, inside the ioctl path that the netlink path returns before reaching. So on 3.27.1 `-p` is *still* ignored, and because the loop never starts, the kernel never receives `NBD_CMD_RECONFIGURE` — the device pauses for `dead_conn_timeout` and then returns `BLK_STS_IOERR` with nothing to reconnect it. That is precisely the silent-failure class this design exists to eliminate, one layer down. (Upstream fixed it on `master` by testing `cur_client->persist` at both sites; the fix postdates 3.27.1.)

**What the script must therefore do, and it must do it as a pre-flight check rather than in the README:**
1. **Require a `nbd-client` built from `master` (or any commit after `eac55e0b06fe`, 2026-08-30).** Not a version floor — a *build* requirement, because **no release can reconnect at all**: 3.27.0 and 3.27.1 have `-T` but gate the persist loop on a field nothing sets, and everything ≤3.26.1 has no `-T` and returns from the netlink path before reaching the loop. The script must check for the capability rather than the number, and must name the required commit in its refusal, because a version floor of "3.27.1 or newer" would pass on exactly the two broken releases.
2. **Pass BOTH `-T <seconds>` and `-p`.** They are not alternatives: `-T` sets how long the kernel waits, `-p` is what actually hands it a new socket when the wait expires. Passing `-T` alone pauses the I/O for the full window and then returns `BLK_STS_IOERR`, which reads exactly like a DittoFS failure and is not one. The script must print both flags in its own output so the transcript says which regime ran.
3. **Assert the mechanism took effect rather than assuming it.** `dead_conn_timeout` is not exposed through sysfs (`drivers/block/nbd.c` creates only `pid_attr` and `backend_attr`), so there is no read-back path — the honest check is behavioural: SIGKILL the server, confirm the guest's I/O **blocks** rather than erroring immediately, and confirm it **resumes** when the server returns. Blocking alone proves only that `-T` reached the kernel; resuming is what proves `-p` did. Both halves are the assertion.
4. **`-p` must be backgrounded in the initramfs**, and the script must say why: the persist branch returns before the daemon call (`nbd-client.c:1571` vs `:1580`), so a foreground `-p` never returns and `switch_root` is never reached. Backgrounding survives `switch_root` — busybox's `delete_contents` unlinks the initramfs files to free memory and does not kill processes.
5. The README must say plainly that **no released `nbd-client` can reconnect** and that the harness needs a `master` build, so nobody "fixes" the harness by adding `-p` back to a release and reporting a mystery I/O error.
6. **The client must run in the init network namespace — and this is the requirement most likely to be violated, because the natural way to run the rig on a development machine is in a container, and a containerized client is deaf to the reconnect notification.** `nbd_mcast_index` sends `NBD_CMD_LINK_DEAD` with `genlmsg_multicast(&nbd_genl_family, ...)` (`drivers/block/nbd.c:2673`), and `genlmsg_multicast` is a `static inline` that hardcodes `&init_net` (`include/net/genetlink.h:530`: `return genlmsg_multicast_netns(family, &init_net, skb, portid, group, flags)`). A client in a non-init netns never receives it, so `persist_conn.link_dead` is never set, the persist loop never calls `reconnect_with_backoff`, and the device behaves exactly as it does with no `-p` at all: **the I/O blocks for the full `dead_conn_timeout` and then fails with `BLK_STS_IOERR`.** Measured on this host, changing only the namespace: in a container netns, a netlink listener on the group received **zero** messages and the read failed at the window; with `--network=host`, the same listener received the message, the client logged `Received link dead notification for device 0` then `Successfully reconnected after 3 attempts`, and the blocked read completed with the bytes written before the crash. Both runs used the identical `master` binary and identical flags, so the namespace is the only variable. The script must verify the namespace before the crash test (refuse unless `readlink /proc/self/ns/net` == `readlink /proc/1/ns/net`) and name `--network=host` in the refusal. Do not attempt a workaround: there is no way to deliver the message into another netns from the kernel side, and `-nonetlink`/`-L` is worse than useless here.

**A working recipe, measured end to end against a live DittoFS on this host, with the verdict it produced.** The ordering is the whole test, and the first two attempts here got it wrong:

```
1. connect WITH BOTH -persist -T 60, then write a pattern and FLUSH it
   (flush matters: without it the pattern is only in the guest's page cache,
    and a read after the crash returns it from cache, proving nothing)
2. SIGKILL the server FIRST, before issuing the read
3. THEN issue the read - it must PAUSE, not fail
4. restart the server inside the window
5. the read completes with the pre-crash bytes
```

Measured result with real DittoFS killed and restarted: the read was still blocked at t=4s, t=8s and t=12s (that is `-T` reaching the kernel); the server returned at t=16s; the read completed `exit=0 elapsed=14s` with **`PRE-CRASH-PATTERN-AAAA`** — the exact bytes written before the crash — and the persist daemon logged `Received link dead notification for device 0` → `Our device, triggering reconnect` → three refused attempts → `Successfully reconnected after 4 attempts`. Blocking proves `-T`, resuming proves `-p`; both in one transcript.

**The two orderings that look right and prove nothing, both actually run here first:**
- **Reading before killing.** The read is served from the guest's page cache and returns in `elapsed=0s`; the kill then lands on an idle device. This passed and meant nothing. Either drop caches (`echo 3 > /proc/sys/vm/drop_caches`) before reading, or — better, and what the recipe above does — kill first so the read cannot be served locally at all.
- **Writing without flushing.** The write is acknowledged into page cache, `dd` reports success, and a later read returns the cached copy. The flush is what makes the bytes the server's problem, and therefore what makes their survival evidence.

A script that reports a pass without a nonzero pause and a completed read after the restart has tested the page cache, not the adapter.

- [ ] **Step 3: Extend `test/nbd/README.md`**

Add the boot and crash sections: prerequisites, the exact `dead_conn_timeout` value used, how to read the verdict, and the honest statement that these require a Linux host with root and cannot run in CI on macOS.

The prerequisites section must state the `nbd-client` requirement plainly, because it is the one thing a reader will otherwise get wrong by using their distribution's package: **no released `nbd-client` can reconnect**, so the harness needs a build from `master` (after `eac55e0b06fe`). Name the two failure modes it avoids — 3.27.0/3.27.1 gate the persist loop on a field nothing sets true, and ≤3.26.1 has no `-T` and returns from the netlink path before reaching the loop — and say that both `-T` and `-p` are required and do different jobs. A reader who tries a release will get a `BLK_STS_IOERR` after the full window that looks like a DittoFS bug; the README is where that is ruled out in advance.

- [ ] **Step 4: Syntax-check both scripts**

Run: `bash -n test/nbd/boot-qemu.sh && bash -n test/nbd/crash-restart.sh`
Expected: no output.

Run: `shellcheck test/nbd/*.sh` if available; fix what it reports.

- [ ] **Step 5: Verify each script refuses cleanly without prerequisites**

Run both on a host lacking QEMU/root and confirm each exits nonzero naming the missing prerequisite — not with a partial run, and not with exit 0.

Also verify the **volume check** specifically, because it is the prerequisite whose absence produces a message that names something else: with a share and adapter present but no `volumes/boot`, the script must fail naming the missing volume, not let the guest boot and fail the handshake with `nbd: unknown export` (which reads as an export-name problem and is not one).

- [ ] **Step 6: Record a run on a Linux host**

This is the step that produces the actual evidence, and it is the point at which the design is falsified or confirmed. Record the transcript in the README's results section: the `dead_conn_timeout` value, the measured restart time, the guest's I/O behaviour, and the host-side `fsck` verdict.

If the host has no KVM, **the script must refuse rather than fall back to TCG.** A TCG run boots and passes, and every timing in it — the dead-connection window, the restart time, the hung-task bound — is meaningless. Report "not run: no KVM" as the honest outcome; a green run under TCG is worse than no run, because it looks like evidence.

Per the spec's §10: **Phase 1 is the first moment the design is falsifiable end-to-end, and it is the right place to stop and reassess if the boot path turns out to need something the protocol surface does not provide.** If it does, stop and report rather than extending the surface ad hoc.

- [ ] **Step 7: Commit**

```bash
git add test/nbd/
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "test(nbd): KVM boot harness and crash-restart rig"
```

---

## Verification Checklist

Run in order; each becomes meaningful at the task named.

- [ ] Task 1 — `go test ./internal/adapter/nbd/ -run TestMagicValues` passes, and fails when a magic is transposed.
- [ ] Task 2 — `TestHandshake_GoNameLengthExceedsOptionData` and `TestHandshake_HugeOptionLengthRejected` pass, and fail when the length checks are removed.
- [ ] Task 2 — `TestHandshake_GoReturnsExportInfoThenAck` asserts `SEND_FLUSH` present and `CAN_MULTI_CONN` absent.
- [ ] Task 2 — `TestHandshake_UnsupportedOptionIsRefused` asserts `NBD_OPT_STRUCTURED_REPLY` is refused with `NBD_REP_ERR_UNSUP`, and no `NBD_FLAG_BLOCK_STATUS_PAYLOAD` is advertised anywhere (the residency-truth gate, spec §7).
- [ ] Task 2 — a restart serves the same export name and size: `TestRuntimeExporter_ResolvesVolumeFile` run twice against the same share returns identical `Size` and `PayloadID` (spec §5.2 obligations 1–2).
- [ ] Task 3 — `TestServe_RefusesNonLoopbackBindWithoutOptIn` passes for `""`, `0.0.0.0`, `::`, and a routable address.
- [ ] Task 3 — `models.DefaultPort("nbd") == 10809` and `AdapterConfig{Type: "nbd"}.Validate()` is nil.
- [ ] Task 3 — `go build ./... && make test-unit` clean.
- [ ] Task 4 — `TestConnection_ReadAtExactEndIsServed` passes, and fails when the bounds check uses `>=`.
- [ ] Task 4 — `TestConnection_OffsetPlusLengthOverflowIsEIO` passes, and fails when the overflow check is removed.
- [ ] Task 4 — `TestConnection_ReadWithFUAFlagSucceeds` passes, and fails when `READ` rejects any non-zero flag word.
- [ ] Task 4 — `TestConnection_ReadZeroLengthIsEINVAL` and `TestConnection_UnimplementedCommandIsEINVAL` pass (the latter answers `EINVAL`, not `ENOTSUP`).
- [ ] Task 4 — `TestConnection_PartialHeaderTimesOut` passes.
- [ ] Task 4 — `TestConn_PanicInHandlerDoesNotKillTheProcess` passes, and fails (by crashing the test binary, not by an assertion) when the `recover` is removed from `Conn.Serve`.
- [ ] Task 4 — `TestConnection_FailedStoreResolutionDegradesAndCounts` passes, and `h.TakeDegraded()` returns 0 on the second call.
- [ ] Task 4 — the `handler` field is an `atomic.Pointer`, and `go test -race` reports no race between `Serve` and `Healthcheck` (drive a probe while a connection is being accepted).
- [ ] Task 5 — `TestConnection_WriteZeroLengthIsEINVAL` passes and the connection is still usable afterwards.
- [ ] Task 5 — `TestConnection_RejectedWriteKeepsTheStreamSynchronized` passes, and fails when the payload drain is removed.
- [ ] Task 5 — `TestConnection_WriteWithFUAIsDurableBeforeReply` passes, and fails when `commit` is moved after the reply.
- [ ] Task 5 — a WRITE refused by POSIX mode bits reaches the client as `ErrEPERM`, not `ErrEIO`. The metadata funnel returns `merrs.ErrAccessDenied` (`pkg/metadata/auth_permissions.go:510`, `:539`), a different code from `ErrPermissionDenied`; the test needs a real metadata service with the volume created as one identity and written as another, or it passes vacuously.
- [ ] Task 5 — `TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO` passes.
- [ ] Task 6 — `TestConnection_WriteZeroesThenReadIsZeros` asserts `st.punches == 1` and `st.writes == 0`, and fails when `WriteZeroes` is changed to a zero-buffer write.
- [ ] Task 6 — `TestConnection_IdleDeadlineClosesASilentConnection` fails when the `Idle` arm is removed, and only when `Idle` is shorter than `Read` (`Idle: 200ms` vs `Read: 5s`). Measured without the arm: `serve still running after 1.5s`.
- [ ] Task 6 — `TestConnection_PayloadReadIsBoundedByRead` fails when the payload arm is removed from `handleWrite` (`Read: 200ms` vs `Idle: 5s`, the inverse fixture). Measured after adding it: exactly one failure, at the test's 1s bound. The reason the test had to be written is the *other* measurement — **before it existed, deleting the same arm left the whole package green**, because the payload read then inherits the 5s header budget and `TestConnection_WriteTruncatedPayloadIsAnError` cannot catch it: equal budgets, and it closes the client, so EOF ends the read before any deadline is consulted.
- [ ] Task 6 — `TestIntegration_TrimFreesAndReadsZero` and `TestIntegration_WriteZeroesDoesNotMaterializeChunks` assert the manifest shrank (`file.Blocks` before/after, read through `GetFile`), not that `DataExtents` shrank. The extent-map assertion is unsatisfiable: `engine.PunchHole` zero-overwrites through the local append log, so the local tier keeps reporting the range, and `DataExtents` over-reports by documented contract.
- [ ] Task 6 — `TestConnection_WriteZeroesWithNoHoleIsAccepted` passes (`NO_HOLE` is a MUST we advertise) and `TestConnection_WriteWithNoHoleFlagIsEINVAL` passes (it is not a `WRITE` flag).
- [ ] Task 6 — `TestConnection_TrimZeroLengthIsEINVAL` and `TestConnection_WriteZeroesZeroLengthIsEINVAL` pin the uniform zero-length answer (`EINVAL`), and each is followed by a valid command on the same connection to prove the stream is still usable.
- [ ] Task 6 — `TestConnection_TrimPastEndIsEINVAL`, `TestConnection_TrimOffsetPastEndIsEINVAL`, and `TestConnection_WriteZeroesPastEndIsENOSPC` pin out-of-range refusal (not clamping), and `TestConnection_TrimIsEPERMOnReadOnlyExport` passes with `SEND_TRIM` still advertised.
- [ ] Task 6 — `go build ./... && go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/...` clean (NOT `make test-unit`, which times out on five unrelated packages on this machine).
- [ ] Task 7 — `test/nbd/run-oracle.sh` exits 0 with identical **normalized** transcripts against `qemu-nbd`, and the script passes `--discard=unmap` (without it the discard step diverges on the oracle's configuration, not on the codec). Measured on `qemu-nbd` 11.1.2: with the flag, a read of the discarded range exits 0; without it, exit 1 and the data survives.
- [ ] Task 7 — the oracle's `qemu-nbd` launch passes `-x export/boot`. Measured: without `-x` the oracle exposes an **empty** export name, so the URI that works against DittoFS fails with `server reported: export 'export/boot' not present` — a divergence that names neither cause.
- [ ] Task 7 — transcripts are normalized before diffing (`sed '/ops;/d'`) and each command's exit status is captured. Measured: `qemu-io`'s timing line changes on every run (two identical runs against the *same* server differ on it alone, so a literal diff can never pass), and a pattern mismatch is reported on stdout while only the **exit code** distinguishes it — the transcript still reads `read 1048576/1048576 bytes at offset 0`.
- [ ] Task 7 (Step 0) — an NBD `FLUSH` persists the file's deferred metadata, not just the block store: reading `rt.GetMetadataStoreForShare(integrationShare)` before and after shows the mtime move. Measured before the fix: `h.commit` left the raw store's mtime **unchanged**, and `FlushPendingWriteForFile(ctx, handle, false)` then returned `flushed=true` and changed it. The four `connection_test.go` FLUSH tests move off `newTestHandler` (nil meta panics: `runtime error: invalid memory address or nil pointer dereference`) onto `newWriteVolumeFixture`, and the EINVAL case still asserts `st.flushes == 0`.
- [ ] Task 7 — the flag-parity check confirms every flag DittoFS advertises is one `qemu-nbd` also advertises.
- [ ] Task 7 — `internal/adapter/nbd/oracle_test.go` skips (not fails) without QEMU, and `run-oracle.sh` treats that skip as a failure; the script prints which tier ran, so a green run says what it checked.
- [ ] Task 8 — the launch line carries **no `hostfwd`**: it forwards host→guest, which is the wrong direction and would try to bind host port 10809 where DittoFS is listening. The guest reaches the host at `10.0.2.2` with no rule.
- [ ] Task 8 — the launch line passes **an explicit `-accel`, selected per host: `kvm` on Linux, `hvf` on macOS.** Measured on this host: `qemu-system-aarch64 -accel help` lists `hvf` and `tcg` only, and `-accel kvm` fails with `invalid accelerator kvm` — so a hardcoded `kvm` makes the boot half unrunnable on the development machine for no reason. `-cpu host` without an accelerator → `Invalid CPU model: host` and the list of valid models, because the default is TCG and `host` has no meaning there; `-cpu max` is the portable substitute. And **no `-drive`/`boot.img`**: the kernel and initramfs come from `-kernel`/`-initrd`, because `-M virt` on arm64 has no BIOS and nothing in the plan creates a bootable disk. `-display none -serial stdio` carries the transcript, which is the only evidence the harness produces.
- [ ] Task 8 — the harness **provisions `volumes/<name>` before the guest boots**, and sets its size after creating it (`ApplyCreateDefaults`, `pkg/metadata/validation.go:492`, sets `Size = 0` for every non-symlink, so `CreateFile` alone yields a zero-length export). Nothing in the repo creates a volume — `grep -rn "volume" cmd/dfsctl/` is empty — so a script that assumes one exists fails at the handshake with `nbd: unknown export`, the same message an omitted `-N` produces.
- [ ] Task 8 — the guest's `nbd-client` line names the export (`-N export/boot`) and mounts `/dev/nbd0`, not `/dev/nbd0p1`. Measured: omitting `-N` makes `nbd-client` send an **empty** export name (`nbd-client.c:1504-1505`), which `RuntimeExporter.Resolve` refuses (`sep <= 0` → `ErrUnknownExport`) — probe result `nbd: unknown export`, reported to the client as a handshake failure that never mentions the flag. And nothing in the path writes a partition table, so a `p1` suffix fails with `no such device` after the handshake and `mkfs` have both already succeeded.
- [ ] Task 8 — `bash -n` clean on both scripts; each refuses cleanly without prerequisites.
- [ ] Task 8 — the crash rig sets `dead_conn_timeout` with `-T`/`--dead-timeout` and fails when netlink is unavailable, rather than silently running the ioctl path where the option is parsed and ignored.
- [ ] Task 8 — the crash rig requires a `nbd-client` **built from `master`** (after `eac55e0b06fe`, 2026-08-30) and checks the *capability*, not a version number: **no release can reconnect**. 3.27.0/3.27.1 gate the persist loop on `persist_mode`, which nothing ever sets true (the `-p` flag sets `persist`); ≤3.26.1 has no `-T` and returns from the netlink path before reaching the loop. Measured across every tag. A version floor of "3.27.1 or newer" would pass on exactly the two broken releases.
- [ ] Task 8 — the crash rig passes **both** `-T <seconds>` and `-p`, because they do different jobs: `-T` sets how long the kernel waits (`wait_for_reconnect`, `nbd.c:1101-1114`, keyed on `live_connections > 0`), and `-p` is the only thing that hands the kernel a new socket (`reconnect_with_backoff` → `netlink_reconfigure` → `NBD_CMD_RECONFIGURE`). The kernel has **no** reconnect of its own (`grep -c nbd_reconnect_socket drivers/block/nbd.c` → 0). Passing `-T` alone pauses the I/O for the full window and then returns `BLK_STS_IOERR`, which reads like a DittoFS failure and is not one. A second process cannot substitute: `nbd_genl_connect` returns `-EBUSY` (`"nbd%d already in use"`) while the device is configured.
- [ ] Task 8 — the window is asserted **behaviourally and in both directions** — SIGKILL the server, confirm the guest's I/O *blocks* rather than erroring (that is `-T`), and confirm it *resumes* when the server returns (that is `-p`) — because `dead_conn_timeout` has no sysfs read-back (`drivers/block/nbd.c` creates only `pid_attr` and `backend_attr`). Blocking alone passes with a broken `-p`.
- [ ] Task 8 — the initramfs script backgrounds `nbd-client -p`, because the persist branch returns before the daemon call (`nbd-client.c:1571` vs `:1580`), so a foreground `-p` never returns and `switch_root` is never reached.
- [ ] Task 8 — `mkfs.ext4` + mount + write + `sync` + unmount + remount in a KVM guest.
- [ ] Task 8 — SIGKILL mid-write → restart → guest I/O resumes inside the dead-connection window; host-side `fsck` clean.
- [ ] The volume is readable as a plain file over NFS (the B1 property, spec §2.2).
- [ ] `docs/guide/cli.md` regenerated; `test/required-tests.sh` unaffected.
- [ ] No commit message names AI tooling or carries a `Co-Authored-By` trailer for it (`test/no-ai-attribution.sh`).
