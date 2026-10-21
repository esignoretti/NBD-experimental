# Why this adapter refuses `NBD_CMD_BLOCK_STATUS`

## The short version

`nbdinfo --map` and `qemu-img map` cannot be used against a DittoFS NBD export, and
`qemu-img convert` therefore copies every byte of a volume instead of only the bytes that
hold data. That is a deliberate refusal, not an unfinished feature, and this page states
what it costs, what would have to be true to lift it, and what to do in the meantime.

The design record is `docs/superpowers/specs/2026-10-08-dittofs-nbd-block-protocol-design.md`
§7. This page is the operator-facing half: §7 says why the gate exists, this says what it
means for you.

## What `BLOCK_STATUS` is for

A block device can be sparse — it can have ranges that were never written, or that were
written and then discarded, and reading them returns zeros. `NBD_CMD_BLOCK_STATUS` (7)
is how a client asks *"which ranges of this device actually hold data?"*, and
`base:allocation` is the metadata context that defines the answer as `allocated` /
`hole` / `zero`.

Everything built on top of it is a copy that avoids copying nothing:

| Tool | What it does with `BLOCK_STATUS` |
|---|---|
| `qemu-img convert` | Skips holes, so a 100 GiB volume holding 8 GiB converts as 8 GiB |
| `qemu-img map` | Prints the allocation map — the tool you use to see what a disk holds |
| `nbdinfo --map` | The same, for any NBD server |
| `qemu-img compare` | Compares allocation first, then contents |
| `nbdcopy` | Sparse-aware copy; falls back to full copy without it |

Without the command, all of them still work — they just read every byte.

## Why it is refused

The command is not advertised, so a conforming client never sends it. If one does anyway,
it falls to the dispatcher's default arm and is answered `EINVAL`, which is the correct
answer for an unsupported command.

The reason is not that mapping is hard. `engine.Store.DataExtents` already exists, already
returns the hole map NFSv4.2 `SEEK` and `READ_PLUS` use, and already has the right polarity:

> The map may over-report data but must never under-report it.
> — `pkg/block/engine/dataextents.go:24`

For `base:allocation` that is exactly right. Over-reporting makes a client copy bytes it
could have skipped, which is **safe**. Under-reporting makes it skip real data, which is
**silent loss**. Same rule, same direction.

The gap is one state. `DataExtents` derives its answer from local-tier bytes plus the
persisted chunk manifest. A range in **neither** — which is what `journal.StateLost`
describes — is absent from the map, and absence means **hole**.

Today that cannot happen. `pkg/block/journal/state.go:15`:

> Nothing holds `StateLost` today: a residency loss the store cannot make durable is
> refused (the interval keeps its local bytes and the caller gets `ErrStateLost`).

But that is a **runtime invariant, not a structural one**. It holds because no code path
currently produces the state — not because the query would be unable to represent it. The
primitive that would make it observable already exists and has no production caller:
`journal.Store.Extents(ctx, id) ([]Extent, error)` at `pkg/block/journal/state.go:111`,
where each `Extent` carries `Off`, `Len`, `State` and `Durable`.

So the argument for refusing is narrow and specific:

> Advertising a residency query whose under-report case is guarded only by a comment is
> the same class of mistake as advertising `SEND_FLUSH` you do not honor.

With structured replies off, this adapter has **no hole-reporting surface at all**, so the
"never under-report" rule is satisfied by construction rather than by care. That is what
correct-by-construction means here.

## What it would take to lift the gate

Two things, and the second is the real work.

**1. Structured replies.** `BLOCK_STATUS` is only legal inside a structured reply, so
`NBD_OPT_STRUCTURED_REPLY` has to be negotiated and the reply path has to grow chunked
replies with a final `NBD_REPLY_FLAG_DONE`. The codec currently has none of this: there is
no `StructuredReply` type, no `NBD_REPLY_TYPE_OFFSET_HOLE`, no
`NBD_REPLY_TYPE_BLOCK_STATUS`, and the only trace of the option is the constant
`OptStructuredReply` at `internal/adapter/nbd/protocol.go:130`, which nothing reads.

**2. The state-aware query.** `BLOCK_STATUS` must be answered from `DataExtents` **unioned
with** `journal.Store.Extents`, so a `StateLost` range reports as **allocated** — forcing
a `READ` that fails loudly — rather than as a **hole**, letting the client skip it. That is
the bounded task the spec describes in §7.3: not "rewrite the state model", but "wire the
existing state primitive into the one surface where its absence is a silent-loss path".

The gate is therefore a checklist, not a judgement call:

- [ ] `journal.Store.Extents` has a production caller.
- [ ] The `BLOCK_STATUS` answer unions it with `DataExtents` and resolves `StateLost` to
      allocated.
- [ ] A test drives a `StateLost` range and asserts the client is told **allocated**.
- [ ] `qemu-img map` against a fixture matches `DataExtents` for a file with known holes.

Until all four hold, advertising the command would trade a documented limitation for an
undocumented data-loss path.

## What to do instead

**Full copy is the only option, and it is correct — just not incremental.** `qemu-img
convert`, `nbdcopy` and `qemu-img dd` all work; they read every byte. For a 1 GiB test
volume that is seconds. For a 1 TiB volume it is the whole volume.

**Filesystem-level backup is unaffected**, because it never asks the block layer. A guest
running `tar`, `rsync`, `restic` or `borg` inside the VM sees a normal filesystem and
copies only the files that changed. The NBD layer only has to serve correct `READ`s, which
it does. This is the reason the limitation matters less than it sounds: the tools that
want `BLOCK_STATUS` are the ones doing *block-level* incremental backup, and the ones doing
*file-level* backup do not.

**Snapshotting is the other half of this, and it is also absent.** An NBD export is a live
view of one volume; there is no point-in-time copy to read from. Taking a backup of a
running VM therefore requires the guest to quiesce its own filesystem (or accept a
crash-consistent copy), which is a property of the guest, not of this adapter. See
"Snapshots" below.

**If you need sparse copy today, use the filesystem.** `qemu-img convert` from an export of
a filesystem that is mostly empty still reads the empty space. Copying the *files* out of a
mounted guest, or using the guest's own backup tooling, avoids that entirely.

## Snapshots

Not implemented, and not on this adapter's roadmap. The design record scopes NBD to a
block-protocol view of one existing volume; point-in-time copies are a metadata and
block-store concern that would have to land in `pkg/metadata` and `pkg/block` first, then be
exposed. Two NBD-specific facts constrain any future design:

- **`NBD_CMD_RESIZE` does not exist in mainline clients.** Neither the Linux kernel's
  request switch (`drivers/block/nbd.c`) nor QEMU's request enum
  (`include/block/nbd.h`, which stops at `NBD_CMD_BLOCK_STATUS = 7`) has a case for it; the
  command and its `NBD_FLAG_SEND_RESIZE` flag belong to an experimental extension branch.
  So a volume cannot be resized under an established client, and a snapshot cannot be
  "exposed as a bigger device" either. A snapshot would have to be a **new export**, which
  means a new connection.
- **The export's size is a snapshot taken at handshake** (`Export.Size`,
  `internal/adapter/nbd/handshake.go`). A volume is an ordinary regular file, so its size
  can change underneath a connection; the transmission phase keeps enforcing the size the
  client was told. A snapshot export would inherit that behaviour, which is correct — but it
  means the client has to reconnect to see a changed size.

## See also

- `docs/superpowers/specs/2026-10-08-dittofs-nbd-block-protocol-design.md` §7 — the
  residency-truth gate, and §11 for the full out-of-scope list.
- `pkg/block/engine/dataextents.go` — the primitive, and the comment that states its
  polarity.
- `pkg/block/journal/state.go` — `StateLost` and the `Extents` query nothing calls.
- `docs/nbd-benchmarks.md` — what the adapter does do, measured.
