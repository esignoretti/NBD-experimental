# SDD ledger — plan: docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md

Worktree: <repo>
Branch: nbd-block-protocol (tracks origin/develop @ 4dae5562)
Spec: docs/superpowers/specs/2026-10-08-dittofs-nbd-block-protocol-design.md (read: yes, 868 lines)
Base for the branch: cc7b34cb (plan + spec corrections committed)

## Pre-flight scan

Eight tasks, all strictly sequential (each task's Interfaces block is how the next
learns its names and types). Shared files and interfaces, task by task:

| Pair | Producer -> Consumer | Found |
|---|---|---|
| 1 -> 2,4,5,6 | protocol.go: Request/Reply/OptionHeader/ReplyHeader/Command/CommandFlags/NBDError + Read*/write* | clean; Task 2 adds option-header decode, Task 1 owns the transmission-phase codec |
| 2 -> 3 | Export{Name,Size,PayloadID,Handle,ReadOnly}, Exporter, NewRuntimeExporter(rt,id), ErrUnknownExport | clean after the id parameter was added to NewRuntimeExporter (see below) |
| 2 -> 4 | Export.Handle is the volume FILE handle (PrepareWrite/CommitWrite need it) | clean |
| 3 -> 4 | pkg/adapter/nbd constructs nbd.NewHandler(exporter, resolver, meta, id, timeouts, draining); Adapter.Handler() accessor | clean |
| 3 -> 7,8 | DefaultPort 10809; the advertised flag set | clean |
| 4 -> 5,6 | BlockStore interface (7 methods), mapError, Handler.serve, Conn | clean |
| 4 -> 5 | common.NormalizeBlockStoreError is EXPORTED by Task 4; Task 5 calls it | sequencing is correct (4 before 5) |
| 4 -> 5,6 | h.commit(ctx, bs, exp) shared by FUA, FLUSH, WRITE_ZEROES+FUA | clean |
| 5,6 -> 7 | the command set the oracle drives | clean |
| 7,8 -> none | scripts only | clean |

Self-consistency per task (its own tests against its own code):

| Task | Found |
|---|---|
| 1 | clean; ErrBadCommand removed so the dispatcher's default branch stays reachable |
| 2 | clean; two NewRuntimeExporter call sites updated for the id parameter; ptr() is test-only |
| 3 | clean; adapter_test.go is package nbd_test (external) so testruntime does not cycle; Handler() accessor added for it |
| 4 | clean; authContext is per command, ShareReadOnly set here and NOT in RuntimeExporter |
| 5 | clean; the payload drain is step 6, before every rejection path |
| 6 | clean; NO_HOLE accepted on WRITE_ZEROES, rejected on WRITE |
| 7 | clean; Python handshake is the primary flag check, nbdinfo optional |
| 8 | clean; bash -n only, needs root + Linux + QEMU, not run in CI |

Plan-mandated defects: none found. The plan mandates no test-that-asserts-nothing and
no verbatim duplication of a logic block. The one duplication the review rubric would
otherwise flag -- a second copy of the commit rule -- is explicitly avoided: Task 4
widens common.CommitBlockStore to an interface instead of copying the rule.

Rulings made during the pre-flight scan:

Ruling: RuntimeExporter.Resolve needs an AuthContext for metadata.Service.Lookup, so
  NewRuntimeExporter gained an `id Identity` parameter. Cost if wrong: two test call
  sites and one production call site change signature.
Ruling: ShareReadOnly is applied in Handler.authContext (client-facing), not in the
  handshake's internal resolution. Cost if wrong: a read-only share's export would be
  refused at handshake rather than rejected per command, which the spec's §4.6
  EPERM row contradicts.
Ruling: the zero-length answer is uniform EINVAL for all four commands (was: TRIM
  no-op success). Cost if wrong: a client that sends a zero-length TRIM expecting
  success gets an error; no known client does.
Ruling: ErrBadCommand is not a codec sentinel. Cost if wrong: nothing -- the
  dispatcher's default branch answers EINVAL either way.

## Task log

Task 1: BASE cc7b34cb, dispatched to `worker` (model cubbit/vllm/mimir), async run d119ad33.
  Brief: .superpowers/sdd/2026-10-08-dittofs-nbd-block-protocol/task-1-brief.md (194 lines)
  Report will be: .superpowers/sdd/2026-10-08-dittofs-nbd-block-protocol/task-1-report.md

### Task 1 — implementer report

Status: DONE. Commit `0f8d7d61` feat(nbd): wire codec for the NBD protocol.
Files: internal/adapter/nbd/protocol.go (284 lines), internal/adapter/nbd/protocol_test.go (124 lines).
Tests: 7/7 pass — TestRequestRoundTrip, TestRequestBadMagic, TestReplyRoundTrip,
  TestOptionHeaderRoundTrip, TestOptionReplyHeaderIs20Bytes, TestMagicValues,
  TestReadRequestTruncated. go build ./... OK, go vet OK, gofmt clean, package lint 0 issues.
TDD evidence: RED = `undefined: Request` build failure before implementing, plus the
  transposition run failing ON THE ASSERTION (`protocol_test.go:110: request magic =
  0x25609514, want 0x25609513` -> `--- FAIL: TestMagicValues`), not a compile error.
  GREEN = all 7 pass after implementation and after restoring the magic.

Ruling: the controller's dispatch message glossed OptionHeader/OptionReplyHeader as
  "Task 2's, not yours". The brief's own Interfaces block assigns them to Task 1, as do
  the plan's Task 1 Produces, the plan's Task 2 Consumes, and the verbatim test in
  Task 1 Step 1. The implementer followed the brief, which is correct: the brief is the
  authority for the task, and a contradiction between brief and dispatch is a controller
  error. No change to the code. Cost if wrong: none -- if Task 2 had wanted to define
  them, the implementer would have hit a duplicate-declaration compile error, which is a
  loud failure rather than a silent one.

Task 1: minor (deferred): repo-wide `make lint` panics on an environmental Go 1.27/1.26
  toolchain mismatch and panics on untouched packages too; package-scoped golangci-lint
  is clean (0 issues). Not a task defect.

### Task 1 — review

Verdict: **Approved**. No Critical, no Important. Reviewer verified every name, type,
  width, and value in the brief's Produces list against internal/adapter/nbd/protocol.go,
  all four encode/decode pairs byte-exact against the NBD header layouts (28/16/16/20),
  and all five magics digit by digit. It also confirmed the two deliberate absences are
  absent and the decision: markers record why.

Task 1: minor (deferred): `CommandFlags` doc comment says "the rest are named so a
  dispatcher can reject them explicitly" but bits 5-15 are unnamed. Comment only.
Task 1: minor (deferred): `ReadOptionReplyHeader`'s decode body is never executed by a
  test (`TestOptionReplyHeaderIs20Bytes` only calls Encode and checks the length). The
  field offsets are correct by inspection against the spec; coverage gap, not a defect.
Task 1: minor (deferred): transmission-flag VALUES are pinned by no Task 1 test.
Task 1: minor (deferred): magic validation is asymmetric — ReadRequest validates,
  ReadReply/ReadOptionHeader/ReadOptionReplyHeader do not. Per the plan (Task 2 step 4
  owns the ihaveoptMagic check); undocumented at the three unchecked sites.

Ruling: the reviewer's stated reason for deferring the transmission-flag gap is
  imprecise, and the real reason is narrower, so the final review does not re-open it.
  The reviewer said "Task 2's flag test plus Task 7's oracle are the downstream checks".
  Both use SYMBOLIC constants, so neither can catch a transposition -- if
  FlagSendFlush and FlagSendFUA swapped values, both assertions still pass, because
  the same wrong constant is on both sides. The gap is nonetheless harmless, for a
  different reason: the four advertised bits (1<<2, 1<<3, 1<<5, 1<<6) are always set
  TOGETHER, so a permutation among them is observationally identical on the wire --
  all four bits are set either way, and a client acting on bit N gets the spec's
  meaning for bit N regardless of which Go name holds that value. A transposition into
  an EXCLUDED bit is caught, because Task 2's exclusion loop asserts on RAW bits
  (`[]TransmissionFlags{1 << 8, 1 << 7, 1 << 9, 1 << 10, 1 << 11, 1 << 12, 1 << 4}`),
  which is the one place in the plan where a flag is pinned numerically. A
  transposition of FlagReadOnly is caught too, by Task 2's symbolic read-write case.
  So the only uncaught class is a permutation within the always-set four, which is
  wire-identical. Cost if wrong: a transposed flag would be advertised with both bits
  set anyway; the failure mode the design exists to prevent (advertising SEND_FLUSH
  when the bit means something else, so QEMU silently elides every flush) requires
  losing the bit, and losing it is caught.

## Controller: plan defect found and fixed before re-dispatching Task 2

Task 2's implementer (run 3594e68d) timed out after 30 min having written only
handshake_test.go. Root cause was a plan defect, not the implementer: the plan
declared types in Task 4 that Tasks 2 and 3 reference, so neither could compile
at its own commit. Task 2's own implementer went looking through Task 3's brief
for the missing type.

Ruling: move Identity to Task 2 — the handshake builds the AuthContext that
metadata.Service.Lookup requires, so NewRuntimeExporter cannot be written
without it — cost if wrong: none, Identity is a two-field struct and its
placement does not affect the wire.

Ruling: Task 3 binds and delegates only (no handler field, no Handler()
accessor, no timeouts accessor, NewConnection is a stub); the wiring becomes
Task 4 step 6, where every symbol it names exists — cost if wrong: Task 4
gains one step and Task 3's NewConnection is replaced rather than filled in,
which the plan now states at both sites.

Ruling: Timeouts lives in Task 4's internal/adapter/nbd, not Task 3 (which
creates no files in that package) — cost if wrong: none, it is an unexported
three-field struct.

Plan fix committed as d1375c53. Briefs 2, 3 and 4 re-extracted. A
forward-reference scan of every task's Go fences against every later task's
declarations now reports NONE.

Salvage: the timed-out run's output, internal/adapter/nbd/handshake_test.go
(415 lines, 12 test funcs, untracked), is byte-identical to the corrected
brief's step-1 block after gofmt. It is kept as Task 2's step 1; the
implementer verifies and formats it rather than rewriting it.

Task 2 BASE: d1375c53

Task 2 re-dispatched (run 27ac69f4-ca73-414d-b63d-195fbf1dbf03), agent worker,
model cubbit/vllm/mimir, async. Brief task-2-brief.md (600 lines, 9 steps).
Salvage instruction carried: verify and keep the existing handshake_test.go
rather than rewriting it.

## Controller: second plan defect — net.Pipe deadlocks the handshake tests

Run 27ac69f4 also timed out at 30 min with only handshake_test.go written. Its
log shows it was NOT confused: it built two throwaway programs under /tmp to
investigate, and its second one printed "DEADLOCK: server write still blocked".

Reproduced independently: net.Pipe is synchronous and unbuffered, so a server
Write blocks until the reader has consumed every byte. TestHandshake_
GoReturnsExportInfoThenAck reads the first option-reply header (20 bytes) plus
the INFO payload (12 bytes) and asserts, leaving the server's ACK write (20
bytes) blocked. The test then waits on a channel the server will never reach,
so it hangs to the Go test timeout instead of failing. A broken handshake
looked like a slow test, which is the worst possible failure mode for the one
test that proves SEND_FLUSH is advertised.

Ruling: replace net.Pipe with a pipe(t) helper returning a connected pair over
a loopback TCP socket, where the kernel buffers the reply — cost if wrong: the
tests are marginally slower and a helper is duplicated in Task 4, versus the
status quo where ten tests can hang.

Plan fix committed as 90cb65bc, which also commits the salvaged
handshake_test.go (438 lines, gofmt'd, 10 pipe(t) call sites). Brief 2
re-extracted at 629 lines and verified byte-identical to the file.

Ruling: the test file is committed with the plan rather than by the
implementer, because it is the plan's own fence verbatim — transcription is
not implementation work — cost if wrong: Task 2's review diff will not contain
the test file, so the reviewer sees production code only. Recorded here so the
final review knows the tests were reviewed as part of the plan.

Ruling: Task 2 re-dispatched with a 60-minute timeout instead of 30. Two
30-minute burns is enough evidence that this task is sized near the limit, and
the remaining work (reply.go, handshake.go, volume.go, volume_test.go) is
comparable to Task 1's ~400 lines — cost if wrong: a longer budget for a task
that turns out to need it anyway.

Task 2 BASE: 90cb65bc

Task 2 re-dispatched a third time (run a0e42b95-0cfa-42c9-b50b-8dd8f44afffe),
60-minute timeout, brief task-2-brief.md (629 lines). Dispatch carried both
fixed defects explicitly so the implementer does not re-investigate either, and
a 15-minute tripwire: reply BLOCKED if still investigating without production
code.

## Task 2: DONE and independently verified

Commit c13b595f "feat(nbd): handshake, option haggling, and export
resolution" (reply.go 48, handshake.go 356, volume.go 198, volume_test.go 80),
then 283105a1 fixing four errcheck hits in the plan-owned test helper.

Independent verification by the controller, not the implementer's report:
- 20/20 tests pass, -race and -count=2 clean; go build ./... , go vet, gofmt
  all clean; full pkg/adapter + internal/adapter/nbd suite passes.
- handshake_test.go verified byte-identical to the brief's step-1 fence.
- Signature present (1 BEGIN SSH SIGNATURE), no-AI-attribution test passes
  against the base.
- The load-bearing assertion bites on the real path: removing FlagSendFlush
  from Export.TransmissionFlags() fails on the ASSERTION at
  handshake_test.go:198 ("NBD_FLAG_SEND_FLUSH not advertised: QEMU turns every
  guest fsync into a silent success without it") and again at :334 for the
  legacy path. Not a compile error.

Advertised flag set verified against the design: HAS_FLAGS | SEND_FLUSH |
SEND_FUA | SEND_TRIM | SEND_WRITE_ZEROES, plus READ_ONLY only when set. The
seven excluded bits are asserted numerically at handshake_test.go:426.

Ruling: the four errcheck hits were in the plan's own test block, not the
implementation, so the fix went into both the file and the plan fence — cost
if wrong: none, the brief is regenerated from the plan so the two cannot drift.

Ruling: handshake_test.go was committed by the controller before the
implementer started (it is the plan's fence verbatim), so it appears in Task
2's review diff as a whole-file addition the implementer did not write. The
reviewer was told this explicitly so the test is still judged on merit rather
than skipped — cost if wrong: a reviewer might attribute a plan defect to the
implementer, which the dispatch note prevents.

Implementer concerns, all three assessed and accepted:
1. RepErrTooBig closes the session. The spec supports this: a server MUST
   simply error inbound options except where it believes the client is a DoS,
   in which case it MAY hard-disconnect. A client that announces an option
   length it cannot deliver is that case, and continuing would mean consuming
   the very length being refused. Carries a decision: comment.
2. Exporter.List failure ends the handshake rather than answering an empty
   list. An empty list tells the client "this server has no exports", which is
   a different and wrong statement. Correct.
3. Client NBD_FLAG_C_FIXED_NEWSTYLE is not validated. The spec says the client
   SHOULD set it, not MUST; the brief asks only for the NO_ZEROES check.

Task 2 review dispatched: reviewer, model cubbit/vllm/mimir, run
9d4f2106-5349-44c3-be39-446baceef1fc, diff review-90cb65bc..283105a1.diff
(870 lines).

Pre-flight for Task 3 done: every symbol Task 3 consumes (ServeHandshake,
RuntimeExporter, NewRuntimeExporter, Exporter, Identity, ErrUnknownExport)
exists in Task 2's output; briefs 3 and 4 re-extracted at 296 and 475 lines.

### Task 2 review: verdict "Needs fixes" — 1 Important, 7 Minor

Diff review-90cb65bc..283105a1.diff (870 lines, 30397 bytes, 2 commits).
Reviewer run 9d4f2106-5349-44c3-be39-446baceef1fc.

Spec compliance: PASSED item by item. The reviewer verified the flag word
against Task 1's constants, the INFO-before-ACK ordering, the 12- and 14-byte
INFO payload widths, the absence of padding after the ACK, the
length-validated-before-allocating ordering, the legacy EXPORT_NAME path's
size+flags+124-zeroes reply, every refusal path, and that Handle comes from
metadata.EncodeFileHandle(file) with no GenerateHandle anywhere in the package.

Important finding: RuntimeExporter.Resolve never checks Share.Enabled, so a
disabled (quiesced) share is still exportable. The reviewer flagged it as
possibly a plan-level omission it could not adjudicate.

I verified it independently rather than taking the reviewer's word, and it is
worse than the reviewer knew: THREE precedents, not one.
- internal/adapter/nfs/mount/handlers/mount.go:114 — refuses with
  MountErrAccess, comment: "the last line of defense when adapters hold a stale
  reference; the runtime Share.Enabled flag is flipped synchronously by
  shares.Service DisableShare before restore begins"
- internal/adapter/smb/handlers/tree_connect.go:119 — refuses with
  StatusNetworkNameDeleted
- internal/adapter/nfs/v4/handlers/helpers.go:226 — refuses a handle for a
  disabled share with NFS4ERR_STALE, i.e. per-operation, not just connect-time
And pkg/controlplane/runtime/shares/service.go:26-29 on the Share struct:
"Disabled shares reject new MOUNT / TREE_CONNECT and in-flight operations."
pkg/controlplane/runtime/shares/share_config.go:281 DisableShare flips the
runtime registry flag synchronously before a restore begins so no adapter
admits a client into the restore window; it then calls InvalidateAuthCache so
established sessions get swept.

Confirmed a genuine plan gap: grep of both the spec and the plan for
"share.Enabled" / "disabled share" / "quiesc" returns nothing.

Ruling: land the gate in Task 2, not Task 4 — cost if wrong: Task 4 would
carry two extra lines and this task would close with a known hole in a file it
owns. Rationale: volume.go is Task 2's file and Resolve IS NBD's only
connect-time gate; the fix is two lines beside a GetShare call that already
happens; Task 4's scope is already the largest in the plan; and an NBD export
is a boot disk, which is exactly the share an operator disables for a restore.

Ruling: the Handle assertion strengthening is ruled IN, not deferred — cost if
wrong: none, it is a one-line assertion that already has the value it needs.
The brief authored the weak assertion verbatim, so per the SDD rules a
plan-mandated finding is the controller's to rule on rather than dismiss. The
brief's central claim for this task is that Handle is the volume FILE's handle
and the test passed for the share root's or volumes/'s handle equally.

Deferred to the ledger as minors (whole-branch review triages):
- Task 2: minor (deferred): RuntimeExporter.List has no coverage at all, and
  it is the only code exercising ListShares + paginated ReadDirectory.
- Task 2: minor (deferred): no test consumes the full GO reply sequence — the
  BLOCK_SIZE reply, the terminating RepAck, and the absence of trailing zeroes
  after it are all unasserted, so a regression emitting 124 zeroes on the GO
  path would pass.
- Task 2: minor (deferred): no read deadline is armed during the handshake, so
  a client that connects and sends nothing holds a connection slot until
  adapter shutdown. Owner (Task 3's timeouts vs this function) is a controller
  call; no brief covers it.
- Task 2: minor (deferred): the client's NBD_FLAG_C_FIXED_NEWSTYLE bit is never
  validated. Spec says SHOULD, not MUST.
- Task 2: minor (deferred): writeSimpleReply is dead code with
  //nolint:unused; a two-line round-trip test would cover it and drop the
  directive.
- Task 2: minor (deferred): an over-long option ends the session rather than
  being drained. The implementer's reasoning is sound and documented.

Reviewer artifact note, no action: the diff shows handshake_test.go as an
8-line modification, not the whole-file addition my dispatch note described.
My note was written for the cc7b34cb base; by 283105a1 the errcheck commit had
already made it a modification. The reviewer handled it correctly.

Fix dispatched as round 1: resumed the original implementer
(a0e42b95 -> revived run abfb2343-9780-45a9-949e-631cd98cec2e), per the SDD
rule that rounds 1-3 resume the original implementer. Carried: the gate in
Resolve and List, the test with RED/GREEN evidence, the Handle assertion, the
plan fence sync in Steps 5 and 6, and an explicit do-not-touch list for the
seven minors.

### Task 2 fix round 1: both findings fixed, plan synced

Implementer: resumed the original run (a0e42b95 -> abfb2343-9780-45a9-949e-
631cd98cec2e), per the SDD rule that rounds 1-3 resume the original
implementer rather than dispatching fresh.

Commits: d9f8e7af "fix(nbd): refuse and omit a disabled share at export
resolution" (volume.go, volume_test.go); df69c170 "docs(nbd): plan the
disabled-share gate and the file-handle assertion" (plan Steps 5 and 6).

Finding 1 ADDRESSED. Resolve: "if !sh.Enabled { return nil, ErrUnknownExport }"
at internal/adapter/nbd/volume.go:101, with a decision: comment at :98-100
explaining the refusal is reported as an unknown export because the export name
is client-supplied and the server must not tell a client whether a share exists
but is quiesced. List: "if !sh.Enabled { continue }" at :180, with the
advertising-consistency rationale at :177-179.

Finding 2 ADDRESSED. volume_test.go:89 is now
"if !bytes.Equal(got.Handle, createdHandle)".

New test TestRuntimeExporter_DisabledShareIsRefusedAndUnlisted
(volume_test.go:97) pins both halves and asserts the enabled state FIRST so it
cannot pass vacuously.

RED evidence, all three removals, all assertion failures:
- gate removed -> volume_test.go:148 "Resolve on a disabled share: got <nil>,
  want ErrUnknownExport"
- List half removed -> volume_test.go:153 "List on a disabled share =
  [/export/boot], want no exports"
- forced handle = root -> volume_test.go:90 "export handle =
  \"/export:01a638f5-...\", want the volume file's handle
  \"/export:6b7c37da-...\""

21 tests pass, -race clean, go build ./... ok, package lint 0 issues. Fix diff
regenerated: review-283105a1..0150e017.diff (3 commits, 19143 bytes).

### The implementer's third concern was real and I fixed the plan for it

The implementer reported: ShareConfig.Enabled has no default and AddShare
copies it verbatim, so its own fixture had been building a DISABLED share all
along and needed Enabled: true. It warned that Tasks 3 and 4 would hit the
same trap.

I verified the claim rather than accepting it, and the warning was half right
in a way that mattered:
- Confirmed pkg/controlplane/runtime/shares/lifecycle.go:548 is
  "Enabled: config.Enabled" with no defaulting anywhere before it, and
  ShareConfig.Enabled's own doc at service.go:146-147 says "Callers pass the
  DB value; AddShare copies it onto the runtime Share."
- Confirmed internal/adapter/testruntime/runtime.go never sets Enabled.
- Confirmed production is NOT affected: the DB column defaults true
  (pkg/controlplane/models/share.go:41, gorm:"default:true;not null") and
  pkg/controlplane/runtime/init.go:314 propagates it with the comment
  "propagate DB Enabled flag so adapter gates read the correct runtime value".
  So this is a fixture-only trap.
- Task 3 is SAFE: it never adds a share. Its fixture is
  serveWithRuntime -> testruntime.NewWithShareStore and nothing more, and
  TestServe_ClientDisconnectDoesNotStopTheAdapter only reads the greeting.
- Task 4 IS affected: its newIntegrationFixture prose said "register a
  metadata store, AddShare with BlockStoreID" with no mention of Enabled, and
  every handler test in Task 4 resolves its export through the gate.

Ruling: patch Task 4's fixture prose in the plan rather than leaving it as a
warning in a report nobody will read at Task 4 time — cost if wrong: two lines
of plan text that restate a real constraint. The patch names the trap, the two
line numbers, why it is fixture-only, and that production is unaffected.

Commit 0150e017 "docs(nbd): note the Enabled fixture requirement for task 4".

Also noted for the record: the implementer's own fixture had been building a
disabled share, and its tests passed because nothing consulted Enabled before
this fix. That is the same class as CLAUDE.md's "when a change starts
populating a field that was always empty, audit the cache keys" — the field
was always false and nothing read it.

Concern carried but NOT acted on (same shape as the other three adapters, and
the implementer flagged it as a scope question): the gate is establishment-only,
so a client that resolved an export before the disable keeps writing. NFSv4's
per-operation check at helpers.go:226 is stricter than NBD's connect-time-only
gate, but NBD has no per-operation handle resolution to hook — the export is
resolved once at handshake by design. Ledgered as an observation.

Scoped re-review dispatched: reviewer, run 78e56b33-610e-464c-a204-1c8d97daf437,
diff review-283105a1..0150e017.diff. I asked it specifically whether the new
GetShare call in List introduced a race the old code did not have (ListShares
returns a name snapshot, then GetShare can fail for a concurrently-removed
share). Checked myself: GetRootHandle at service.go:459 has the identical
shape and returned an error on the same path before this change, so the
behavior is unchanged rather than new.

### Task 2 CLOSED: fix accepted

Scoped re-review (run 78e56b33-610e-464c-a204-1c8d97daf437) verdict: "Fix
accepted". Both findings ADDRESSED at the named seams; no new breakage.

The reviewer independently ruled out vacuity on both halves, which is the part
worth recording:
- The Resolve gate cannot be reached vacuously: GetRootHandle (:84) and
  GetShare (:88) already resolve the same name, so :101 is the only branch
  whose outcome changes when Enabled flips.
- The disabled-half Resolve assertion cannot pass on a foreign error: the
  immediately preceding call at volume_test.go:134 resolves the SAME name on
  the SAME fixture with err == nil, so the only delta is SetEnabledForTesting
  at :143.
- The disabled-half List assertion cannot pass vacuously: the enabled-half call
  at :137-140 first asserts exactly [/export/boot] on the same fixture, which
  rules out the alternate explanation (a missing volumes/ directory would hit
  the IsNotFoundError skip and yield zero names either way).
- All three RED messages are internally consistent with the code that produces
  them, so the RED claims are credible rather than merely asserted.

It also confirmed the List race I asked about is NOT new: GetShare
(service.go:406-412) and GetRootHandle (:459-467) both read the same registry
map under the same RLock and fail only on absence, so the diff moves the lookup
one step earlier without adding a failure mode.

It corrected the implementer's concern #1 for the record: the plan has
ShareConfig literals at only :877 and :951, both in Task 2, and fixtures
elsewhere that omit Enabled never resolve through RuntimeExporter (the only
NewRuntimeExporter call sites are volume_test.go:69,133 and
handshake_test.go:434). So my Task 4 patch covers the whole live surface.

Out-of-scope observations, ledgered as deferred minors:
- Task 2: minor (deferred): List performs one extra locked registry read per
  share. Handshake-time only.
- Task 2: minor (deferred): the gate is resolution-only, so a client that
  resolved before DisableShare keeps writing through the restore window. Same
  shape as all three other adapters; the plan's Step-5 paragraph states it.
  Scope question for the whole-branch review.

TASK 2 COMPLETE. Commits: c13b595f (implementation), 283105a1 (test helper
errcheck), d9f8e7af (disabled-share gate), df69c170 (plan sync),
0150e017 (Task 4 fixture prose). Plan commits before it: 8456999e (spec),
cc7b34cb (protocol corrections + plan), d1375c53 (forward-reference fix),
90cb65bc (net.Pipe fix + salvaged test file).

### Task 3 pre-dispatch: three defects found in the plan's own tests (fixed)

Before dispatching, I read Task 3's step-1 test fence and checked each
assertion against the shared base adapter. Three of them assert capabilities
the code does not have. All three were confirmed by RUNNING the cases, not by
reading.

Defect 1 — the listener tests assert Port() != 0 after binding an ephemeral
port. pkg/adapter/base.go:598 is "return b.Config.Port", i.e. the CONFIGURED
port, and grepping for any assignment to Config.Port in pkg/adapter/ returns
nothing. So a correctly bound listener reports 0 and the test fails on correct
code. Worse, TestServe_ClientDisconnectDoesNotStopTheAdapter then dials
net.JoinHostPort("127.0.0.1", strconv.Itoa(a.Port())) = "127.0.0.1:0", which
cannot connect: probe output "dial tcp 127.0.0.1:0: connect: can't assign
requested address". Fixed by using GetListenerAddr (pkg/adapter/base.go:575),
which returns b.listener.Addr().String() and blocks on listenerReady — so it is
both the correct assertion and the correct thing to dial.

Defect 2 — TestServe_AllowsLoopbackBind includes a bare "::1". BaseAdapter
builds its listen address as fmt.Sprintf("%s:%d", BindAddress, Port)
(pkg/adapter/base.go:216), so "::1" yields "::1:0" and net.Listen rejects it
with "too many colons in address". Probe output:
  Listen("::1:0"       ) -> ERROR: listen tcp: address ::1:0: too many colons in address
  Listen("[::1]:0"     ) -> ok, bound [::1]:58438
That is a property of the shared base adapter, not something Task 3 changes, so
the test now uses "[::1]" — the spelling that binds — rather than asserting one
the code lacks. Both spellings pass netutil.IsNonLoopbackHost as loopback
(pkg/netutil/netutil.go:16-18), so the gate admits either; only the listen
address differs.

Defect 3 — the import instruction said to add "io, strconv". strconv was only
used by the dial in defect 1, so the corrected fence needs io only.

Ruling: fix the tests in the plan rather than let the implementer discover
them — cost if wrong: three test bodies differ from what a reader of the
original plan would expect, in the direction of passing. Rationale: these are
the plan's own step-1 fence, the implementer's instructions are to write the
test as given, and a test that fails on correct code would have burned a
30-minute run and then looked like an implementation bug. The alternative
(change BaseAdapter to use net.JoinHostPort) was rejected: I probed it and it
REGRESSES the bracketed form, because JoinHostPort("[::1]", p) yields
"[[::1]]:p" — net.Listen rejects that with "missing port in address". A correct
fix needs the bracket-stripping logic netutil.IsNonLoopbackHost already has, and
that is a change to shared code serving three adapters, well outside this
task's scope. Recorded here so the "should BaseAdapter use JoinHostPort?"
question does not get re-litigated.

Probe evidence for the rejected alternative:
  host="::1"   naive="::1:12049"   join="[::1]:12049"   ok    <-- DIFFERS
  host="[::1]" naive="[::1]:12049" join="[[::1]]:12049" ERR   <-- DIFFERS
i.e. JoinHostPort fixes one spelling and breaks another; only bracket-stripping
fixes both.

Commit 99f0e834 "docs(nbd): fix three defects in task 3's plan-authored tests".
Brief 3 re-extracted (311 lines). BASE for Task 3 = 99f0e834.

### Task 3: implementer found a real self-contradiction in the brief

It stopped and asked rather than choosing, which is the right call.

The contradiction: Step 4 item 2 requires
"if a.runtime == nil { return errors.New(...) }" (and Task 4's Step 6 preserves
that line), while Step 1's verbatim test file contains
TestServe_ClientDisconnectDoesNotStopTheAdapter, which does cfg.Port = 0;
a := nbd.New(cfg); go a.Serve(ctx) with NO SetRuntime call and waits only on
ListenerReady/5s — never selecting on done. With the check in place Serve
returns immediately, never binds, and the test fails at adapter_test.go:171
"listener was never ready" after 5s.

I verified it independently rather than accepting the report:
- The test's body in the plan's Task 3 fence has zero occurrences of
  SetRuntime (grep -c returns 0).
- The brief's own prose says "Serve needs a runtime once it gets past the gate,
  so the listener tests use the real fixture" — but that note was applied only
  to serveWithRuntime, not to this test.
- The helper already in the fence does exactly what the test needs:
  serveWithRuntime calls testruntime.NewWithShareStore(t), a.SetRuntime(rt),
  starts Serve on a cancellable ctx, and registers a t.Cleanup that cancels and
  waits for Serve to return with its own 5s guard.
So the fix is smaller than the implementer's proposal: use the existing helper
instead of adding two inline lines.

Ruling: approve fixing the test, not the check — cost if wrong: a brief-authored
test body differs from the original in the direction of passing, and Serve keeps
a nil-runtime refusal that the bind-gate tests do not need. Rationale: the
nil-runtime check is an explicit numbered requirement that Task 4's Step 6
preserves, and it is a real guard (Serve with no runtime would otherwise nil-deref
in the wiring Task 4 adds); the test predates the check, so the test is the
stale artifact. The alternative (drop the check) would leave a latent nil
dereference for the sake of a test that was never about the runtime.

Also ruled: use serveWithRuntime rather than inline lines, and drop the now
redundant trailing cancel()/<-done pair — cost if wrong: none, the helper's
cleanup already cancels and waits. One construction path for every listener
test is worth more than the two lines saved, and it keeps the test focused on
what it actually asserts (a per-connection failure is not fatal to the adapter).
Flagged the one subtlety to preserve: the 300ms select on done asserting Serve
did NOT return must stay, and the test must not be reordered so the cleanup's
read could precede it.

Also required: the same edit in the plan's Task 3 Step 1 fence, with a comment
in the fence explaining why this test needs the helper, so the brief regeneration
cannot reintroduce the contradiction.

Explicitly NOT allowed: weakening the bind gate, or adding SetRuntime to the
bind-gate tests (they must keep asserting the gate refuses before the runtime is
consulted).

### The `make test-unit` five timeouts are environmental — verified, not assumed

The implementer ran `go test -race ./...` (step 9) and it reported 139 ok and 5
FAIL, all "panic: test timed out after 10m0s":
  internal/adapter/nfs/v3/handlers   609.383s
  internal/adapter/smb/handlers      606.250s
  pkg/block/chunker                  600.632s
  pkg/block/engine                   601.395s
  pkg/block/middleware/encryption    601.361s

None of the five is a package Task 3 touches (pkg/adapter/nbd,
pkg/controlplane/models/adapter.go, cmd/dfs/commands/start.go). I verified they
are contention rather than regressions by stashing the implementer's work and
running the same packages on the CLEAN BASE (99f0e834) with -p 1:
- pkg/block/engine still timed out on the unmodified base (241s), stack in
  lukechampine.com/blake3/guts.CompressEigentree under -race.
- pkg/block/middleware/encryption PASSED serially in 198s, having failed at
  601s in parallel.
So the failure moves with parallelism, not with the diff.

Root cause shape: the nfs/v3/handlers stack is stuck in SQLite AutoMigrate
(pkg/controlplane/store/gorm.go:433 -> gorm.io/gorm.(*DB).AutoMigrate) reached
from testing.NewHandlerFixtureWithStore (fixtures.go:138). Many packages each
create their own SQLite under -race; running them concurrently on this machine
starves them past the 10-minute deadline.

MY MISTAKE, recorded so it is not repeated: I ran `git stash push --include-
untracked` on the worktree WHILE THE IMPLEMENTER WAS STILL RUNNING. That
temporarily removed pkg/adapter/nbd/ from under an active agent. I restored it
immediately (git stash pop) and verified all six files intact (adapter.go 153,
adapter_test.go 202, config.go 138, healthcheck.go 30, healthcheck_test.go 34
lines) and `go test ./pkg/adapter/nbd/` passing in 1.65s. Nothing was lost, but
that is a hazard to avoid: never mutate the working tree a running implementer
owns. The correct way to test a clean base is a separate git worktree, not a
stash.

Ruling: treat the five timeouts as a known-environmental result and require
instead `go test -race ./pkg/adapter/nbd/... ./pkg/controlplane/models/...
./cmd/...` plus a -p 1 result for any changed package — cost if wrong: a real
regression hiding inside one of the five could go unnoticed in this task. I
judged that acceptable because none of the five imports the changed packages
(Task 3 changes are pkg/adapter/nbd, models/adapter.go, start.go) and the
engine timeout reproduces on the base.

Also told the implementer: do not run a second full ./... (16 minutes, same
result), and that timeout(1) does not exist on this machine — use
`go test -timeout 240s`.

### Follow-up: the implementer started waiting on me, and the plan's step 8 is wrong

The implementer saw my stash appear, correctly did not touch it, and began
sleeping in a loop waiting for "the supervisor's verification to finish" —
by then I had already popped the stash and finished the verification. Steered
it to stop waiting, with the explicit state (stash list empty, work present,
tests passing) so it does not try to pop anything.

Also caught: the plan's Task 3 step 8 says
  go run ./cmd/dfs docs --output docs/guide/cli.md
but CLAUDE.md:86-96 says the generator is `go run ./cmd/gendocs` ("generated
from the Cobra command trees by cmd/gendocs. Never hand-edit it"). The plan's
invocation is wrong.

Checked whether regeneration is needed at all: Task 3's diff to
cmd/dfs/commands/start.go touches no cobra command definition and no flag —
`git diff cmd/dfs/commands/start.go | grep -E "^\+.*(cobra|Flags\(\)|Command\{|Use:)"`
returns nothing. It adds `createNBDAdapter` and a switch case, which is adapter
construction, not CLI surface. And docs/guide/cli.md has zero mentions of "nbd"
today while remaining unmodified. So an unchanged cli.md is the CORRECT
outcome here, not a missing step.

Told the implementer to say so plainly rather than forcing a diff, and to use
`go run ./cmd/gendocs` (not the plan's command) if it runs the generator at all.

DEFERRED until Task 3 commits (the implementer owns the plan file right now —
it is editing the Step 1 fence for the SetRuntime fix, so a controller edit
would collide):

Fix plan line 1323-1326, Task 3 step 8. Current text:
  **Step 8: Regenerate the CLI docs**
  Run: `go run ./cmd/dfs docs --output docs/guide/cli.md` (check the exact
  invocation in CLAUDE.md's "Frequent commands" before running).
  Expected: `docs/guide/cli.md` updated; `git diff --stat` shows it.

Two defects: the command is wrong (CLAUDE.md:86 says `go run ./cmd/gendocs`),
and the expectation is wrong — this task changes no CLI surface, so an
unchanged cli.md is the correct outcome and "Expected: updated" invites an
implementer to manufacture a diff. Replace with the gendocs invocation and an
expectation that states either outcome is fine, naming the check
(`git diff cmd/dfs/commands/start.go | grep -E "^\+.*(cobra|Flags\(\)|Command\{|Use:)"`
returns nothing → no CLI surface changed → cli.md legitimately unchanged).

Same class as the earlier Task 3 test defects: an instruction that asserts a
result the code does not produce.

### Task 3 CLOSED (implementation). Review dispatched.

Commit abc387b6 "feat(nbd): adapter lifecycle, control-plane registration, bind
gate" — SSH-signed (signature check = 1), tree clean, 8 files, 609 insertions.
Files: pkg/adapter/nbd/{adapter.go 153, adapter_test.go 202, config.go 138,
healthcheck.go 30, healthcheck_test.go 34}, cmd/dfs/commands/start.go +41,
pkg/controlplane/models/adapter.go +7/-2, plus the plan fence fix for the
SetRuntime contradiction.

Verified independently:
- `go test -race ./pkg/adapter/nbd/... -count=1` -> ok, 5.145s.
- The bind gate is exactly as specified and refuses BEFORE the runtime check:
  pkg/adapter/nbd/adapter.go:92
    if netutil.IsNonLoopbackHost(a.config.BindAddress) && !a.config.AllowRemoteBind
  with an error that names allow_remote_bind and explains the threat in the
  message itself ("NBD has no authentication and no per-path scoping, so a
  remotely reachable export hands every byte of the volume to anyone who can
  reach the port"). The empty-bind rationale is a comment at :87-91, as asked.
- :99-101 is the runtime check, after the gate. Correct order preserved.
- The implementer also fixed `conn.Close()` -> `_ = conn.Close()` in the test
  fence (errcheck), same as it did in Task 2.

Implementer concerns, all four assessed:
1. The brief self-contradiction — escalated, ruled on, fixed. Already ledgered.
2. The five timeouts — environmental, verified. Already ledgered.
3. docs/guide/cli.md regenerated with NO diff — CORRECT, and it reported this
   instead of staging an empty change. It also found the plan's step 8 named a
   nonexistent command. Ruled: fix the plan (commit a0cc7e20) rather than leave
   it, because the same step is read again if the task is ever re-run.
4. createNBDAdapter has no direct test. Ruled: ACCEPT, defer to the ledger.
   Rationale: the function is pure config plumbing over cfg.GetConfig() with a
   port default; its only branches are four `parsedConfig["x"].(T)` type
   assertions, and the meaningful half (that models.DefaultPort("nbd") returns
   10809 and Validate accepts "nbd") IS tested by
   TestControlPlane_DefaultPortKnowsNBD. A control-plane fixture to cover the
   map plumbing would cost more than the coverage is worth at this layer; the
   end-to-end path is exercised by Tasks 7-8.
   -> ledger: "Task 3: minor (deferred): createNBDAdapter has no direct test."
5. DefaultUID/DefaultGID exported beyond the stated Interfaces contract. Ruled:
   ACCEPT as additive and correct — config.go:19-25 documents them, :121/:124
   apply them, and applyDefaults needs named constants rather than literals at
   two call sites. Exported is the right call for values an operator may want to
   reference; the plan's Interfaces list was not exhaustive and did not claim to
   be. No action.

Review package: review-99f0e834..abc387b6.diff (1 commit, 31409 bytes),
generated BEFORE committing my plan fix so the range is exactly the
implementer's commit. Brief 4 extracted (477 lines) after the fix so Task 4's
brief carries the corrected docs/timeout guidance.

### Task 3 review: 2 Important + 7 Minor. I verified the Importants myself.

Reviewer run ad23c847-50c8-49b4-9e1f-cee206511b8d, verdict "Needs fixes".
Spec compliance PASSED item by item (all Files, all Interfaces including the
deliberate ABSENCE of a handler field, all ten steps, and the three known
defects confirmed not reintroduced). It also confirmed the gate-ordering test
"can only pass for the right reason": TestServe_RefusesRemoteBindEvenWithNoRuntime
asserts the error does NOT contain "SetRuntime", so inverting the order fails on
that assertion.

IMPORTANT 1 — cfg.Port = 0 does not mean ephemeral. CONFIRMED EMPIRICALLY.
pkg/adapter/nbd/config.go:102 is `if c.Port <= 0 { c.Port = DefaultPort }`, and
adapter.go:43 runs applyDefaults before NewBaseAdapter sees the port. Probe:
  Config{Port: 0} -> a.Port() = 10809 (DefaultPort = 10809)
  "0 was REWRITTEN to 10809 -> the tests bind a FIXED port"
So the three listener tests bind 10809, not an ephemeral port, and their
comments claim otherwise.

CONSEQUENCE PROVEN, not inferred — I occupied 10809 with a listener and ran them:
  --- FAIL: TestServe_AllowsLoopbackBind/127.0.0.1
      adapter_test.go:116: Serve returned before the listener was ready: failed to
      create nbd listener on port 10809: listen tcp 127.0.0.1:10809: bind:
      address already in use
  --- FAIL: TestServe_ClientDisconnectDoesNotStopTheAdapter
      adapter_test.go:171: listener was never ready
This is a plausible condition, not a hypothetical: 10809 is this project's own
NBD port, so a developer running the server with NBD enabled holds it.

MY OWN DEFECT: the "ephemeral" comments came from MY plan edits, not the
implementer's invention. Plan lines 1169 ("ephemeral: the test must not need a
fixed free port") and 1181 ("Port() is legitimately 0 whenever the test asked
for an ephemeral one") are mine, and the implementer copied them faithfully.
I introduced a claim I never tested. The implementer is not at fault here.

IMPORTANT 2 — config.go:131-134's validate() comment describes an unreachable
state. It says "Port 0 is valid: it asks the OS for an ephemeral port, which is
what the lifecycle tests use", which contradicts Config.Port's own doc at :64
("0 means the default (10809)") and cannot occur because applyDefaults runs
first. Consequence: the `c.Port < 0` arm is dead too (a negative becomes 10809
at :102), leaving only `> 65535` reachable — which is exactly what
TestNew_PanicsOnOutOfRangePort covers.

SECOND DEFECT THE REVIEWER MISSED — serveWithRuntime's done channel is read
twice. The helper's t.Cleanup reads `done`, and two call sites read it in the
test body as well (adapter_test.go:116 in the select on ListenerReady, and
:180 asserting Serve did not return). On the failure path the body drains the
buffered channel first, so the cleanup's select blocks the full 5 seconds and
then reports a BOGUS second error:
  adapter_test.go:116: Serve returned before the listener was ready: ... address already in use
  adapter_test.go:92:  Serve did not return after context cancellation
Serve DID return; the body already consumed the value. The observable harm is a
misleading 5-second stall plus a false second failure, on the path an operator
hits when the port is busy. Not Critical (test-only, no production impact) but
worth fixing while the file is open.

Rulings on the fixes:
(a) Issue 1: fix the TESTS with a freePort helper (bind :0, read the port,
    close) — probe-verified that a NONZERO port survives applyDefaults
    (freePort 50990 survived). Rejected the alternative of changing applyDefaults
    to treat 0 as ephemeral: that would make an explicit 0 bind a random port in
    production, which is a behaviour change to serve a test, and it contradicts
    Config.Port's documented meaning. The repo already has this helper's shape at
    pkg/controlplane/api/tls_test.go:192. Safe here because adapter_test.go has
    zero t.Parallel calls, so there is no window between close and rebind.
    Cost if wrong: a rare flake if another process grabs the port in the
    microseconds between; the current code fails deterministically whenever
    10809 is busy, which is worse.
(b) Issue 2: correct the comment and drop the dead `c.Port < 0` arm, stating the
    reachability plainly. Cost if wrong: a future caller invoking validate()
    directly with a negative port would not catch it — but validate() has exactly
    one call site (adapter.go:45, immediately after applyDefaults), so the arm
    cannot fire.
(c) The done-channel double read: give serveWithRuntime a cleanup that does not
    consume the body's value, or have the body not read `done` and instead
    observe ListenerReady only. Cleanest: keep the channel for the body and let
    the cleanup cancel-and-wait without reading it — i.e. the cleanup should wait
    on a separate signal. I will let the implementer choose between (i) a second
    channel closed by the goroutine, or (ii) the cleanup cancelling and then
    selecting on done only if the body has not consumed it (not expressible
    simply), so (i) is the expected answer: `go func() { err := a.Serve(ctx);
    close(served); done <- err }()` with the cleanup waiting on `served`.
    Cost if wrong: one extra channel in a test helper.
(d) Minors 3, 4, 5, 6, 8: DEFER to the ledger except where Task 4 is affected.
    - Minor 3 (unread mapstructure tags for uid/gid/timeouts/max_message_size):
      DEFER but flag to Task 4 — Task 4's wiring consumes Config.Timeouts via
      timeouts() and Config.UID/GID via Identity, and start.go does not read
      them, so the config file's values for those keys are silently ignored
      today. Task 4 should either read them in createNBDAdapter or the tags
      should go. Ledger it as a cross-task note, not a Task 3 fix, because the
      brief's Step 6 specifies exactly the four keys it reads.
    - Minor 4 (UID 0 unexpressible because 0 doubles as unset): DEFER. Real, but
      resolving it (pointer/optional field) is a Config shape change that would
      ripple into Task 4's Identity wiring; better decided once, with Task 4's
      usage in hand. Ledger.
    - Minor 5 (UpdateAdapter preserves a running listener when the address is
      unchanged, so flipping allow_remote_bind true->false does not re-run the
      gate): DEFER, pre-existing and outside this diff. Worth a whole-branch
      note because "allow_remote_bind: false in the row" is not a guarantee
      about a live adapter.
    - Minor 6 (the "" subtest renders as #00 in -v output): DEFER, cosmetic.
    - Minor 8 (the SetRuntime comment's stated reason is wrong — Runtime.Metrics()
      is nil-receiver-safe at runtime.go:867-872 and RecordConnAccepted returns
      early on nil at instruments.go:266-269): DEFER, comment-only, but it is a
      false justification attached to a correct call, so the final review may
      want it.

CORRECTION to my own note on Minor 8: I repeated the reviewer's claim that the
comment "a base adapter with a nil Registry would panic on the first accepted
connection" is factually wrong. Reading the committed file, that sentence does
not exist. adapter.go:59-72 says only:
  // SetRuntime stores the control-plane runtime. It panics on any other type,
  // matching [adapter.BaseAdapter.SetRuntime]'s documented contract.
  //
  // decision: the export resolver and the wire handler are built in Serve, not
  // here. ... building them in Serve is what makes a bad configuration or a
  // missing runtime a startup error rather than a failure on the first
  // connection.
  func (a *Adapter) SetRuntime(rt any) {
      r, ok := rt.(*runtime.Runtime)
      ...
      // The base adapter keeps its own reference for the shared accept loop,
      // which reaches it for connection metrics.
      a.BaseAdapter.SetRuntime(rt)
      a.runtime = r
  }
The "which reaches it for connection metrics" line is TRUE (pkg/adapter/base.go:359
and :387 both call b.Registry.Metrics().Record*), and the nil-safety the reviewer
cited (runtime.go:867-872, instruments.go:266-269) makes the CALL safe, not the
comment wrong. The reviewer appears to have attributed a sentence to this file
that belongs to its own reasoning. No action; Minor 8 withdrawn.

This is the second time in this session a reviewer's stated REASON was imprecise
while its underlying observation was worth checking (the first was Task 1's
transmission-flag note). Both times the right move was to verify the claim
against the file rather than either accept or dismiss it.

## Task 3 — fix round 1 (review verdict: Needs fixes)

Ruling: fix the tests with a `freePort` helper rather than change `applyDefaults`
to treat 0 as ephemeral — cost if wrong: a rare flake in the microseconds between
freePort releasing the port and the adapter rebinding it, versus the status quo
where the listener tests fail deterministically whenever 10809 is occupied. The
alternative is worse because it would make an explicit `Port: 0` bind a random
port in production, contradicting Config.Port's own documented meaning
(config.go:64, "0 means the default (10809)").

Ruling: correct validate()'s comment and drop the dead `c.Port < 0` arm — cost if
wrong: a future caller invoking validate() directly, without applyDefaults first,
would not catch a negative port. There is exactly one call site (adapter.go:45).

Ruling: the helper's cleanup waits on its own `served` channel instead of reading
`done` — cost if wrong: none; the defect was a phantom second error on the
failure path, where the test body had already drained the buffered channel.

Ruling: the three "ephemeral" comments originated in MY plan edits (plan lines
1169, 1181), not in the implementer's work, and the same defect was corrected in
the plan fence in the same commit so a regenerated brief cannot reintroduce it —
cost if wrong: none.

Verification of the fix, run by the controller: with 10809 held by a separate
listener the three listener tests now PASS (they failed before the fix with
`bind: address already in use`); `go test -race ./pkg/adapter/nbd/...` is green;
the plan fence is byte-identical to the committed test file.

## Task 3 — a defect of mine that the fix's own report surfaced

`go test ./cmd/...` fails TestDocsCommandReferences on
docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md:1352, a sentence I
wrote in commit a0cc7e20: "There is no `dfs docs` subcommand." The checker
(cmd/gendocs/helpref_test.go:80 findInvocations) is a line scanner with a
prosePrefix guard for English function words and no negation awareness, so it read
my negation as a command reference.

Probed, rather than assumed: the OLD wording ("`go run ./cmd/dfs docs --output
docs/guide/cli.md`") produced ZERO invocations — the checker misses `./cmd/dfs`
paths — so my "correction" is what introduced the failure. Reworded to "An earlier
draft of this step named a `docs` subcommand on the `dfs` binary; the CLI has
none." (probed: 0 invocations). `go test ./cmd/gendocs/` is green.
Committed as 3d974a8f.

Lesson: a checker that scans docs/ also scans the plan and spec I am writing in
docs/. Prose ABOUT a nonexistent command is as visible to it as a real invocation,
and a backticked command name is what makes it visible.

## Task 3 — deferred minors (for the whole-branch review to triage)

- createNBDAdapter (cmd/dfs/commands/start.go:1037-1050) reads only
  bind_address, allow_remote_bind, default_export, max_connections; Config.Timeouts,
  MaxMessageSize, UID and GID carry mapstructure tags nothing honors. Task 4
  consumes timeouts() and Identity, so decide there: read them or drop the tags.
- UID == 0 is unexpressible (config.go:121-126 treats 0 as "unset"), so "run as
  root" cannot be configured even deliberately.
- pkg/controlplane/runtime/adapters/service.go:192 UpdateAdapter preserves a
  running listener when sameListenAddr holds, so flipping allow_remote_bind
  true->false on a bound adapter does not re-run the gate. Pre-existing and
  outside this diff; worth noting because "false in the persisted row" is not a
  guarantee about a live adapter.
- The "" subtest renders as #00 in -v output.
- The gate is establishment-only: a client that resolved an export before the
  share was disabled keeps writing. Same shape as the other three adapters'
  connect-time gates; NBD resolves its export once at handshake by design.

## Task 3 — scoped re-review of the fix (verdict: all ADDRESSED)

All four findings ADDRESSED at the cited lines, no Critical or Important new
breakage. Verified independently by the controller rather than taking the
report's word:

  go test -race -count=1 ./pkg/adapter/nbd/...   -> ok
  go test ./cmd/gendocs/ -run TestDocsCommandReferences -> ok

Reviewer's one Minor, which I ruled IN rather than deferring: the freePort
comment's conclusion ("No t.Parallel in this file, so there is no window between
releasing the port and the test rebinding it") is stronger than its premise
supports. The window is unconditional — freePort's deferred Close releases the
port before it returns — and the absence of t.Parallel only removes competitors
inside this package; under `go test ./...` other packages run concurrently.
Ruling: fix the comment now rather than ledger it — cost if wrong: none, it is a
comment. Rationale: Task 4 copies this helper into its own listener tests, so an
overclaiming comment propagates. This is the THIRD instance this session of a
comment asserting more than the code guarantees (after the "ephemeral" claim and
validate()'s unreachable-state note), which is now a recognisable pattern worth
watching for in Tasks 4-8: when a test helper or comment explains WHY something
is safe, the reason must be the actual reason.

Committed 91ff81d0 (file + plan fence together, so a regenerated brief matches).
Reviewer's residual risk, recorded: it had no shell, so its verification was
static reading plus the implementer's report; the controller ran both commands.

## Task 4 — pre-dispatch audit: seven plan defects found and fixed (commit 8f0f64c3)

Read the brief against the repo BEFORE dispatching, because Tasks 1, 2 and 3 each
burned an implementer run on a plan defect. Seven found, all fixed in the plan;
brief re-extracted to 625 lines.

1. DATA RACE (Important). Adapter.handler was a plain `*nbd.Handler`, written by
   Serve on the accept-loop goroutine (pkg/controlplane/runtime/adapters/service.go:611
   starts adp.Serve inside `go func()`) and read by Healthcheck from an HTTP
   handler (internal/controlplane/api/handlers/adapters.go:240 -> Runtime.AdapterChecker
   -> adp.Healthcheck). Reproduced the exact shape under -race: WARNING: DATA RACE
   between the Serve write and the Healthcheck read. Now `atomic.Pointer[nbd.Handler]`;
   needs `sync/atomic` in pkg/adapter/nbd/adapter.go.

2. DROPPED LINE (Important). Step 6's SetRuntime replacement omitted
   `a.BaseAdapter.SetRuntime(rt)`, which Task 3's committed body has and which the
   shared accept loop needs for connection metrics (pkg/adapter/base.go:359 and
   :387 both call b.Registry.Metrics().Record*). Probe: b.Registry is a CONCRETE
   *runtime.Runtime, and both Metrics() and the Record* methods are nil-safe, so
   the omission degrades to silently dropping connection metrics rather than
   panicking — quieter, and therefore worse.

3. NO RECOVER IN THE CONNECTION PATH (Important, process-killing). The base
   adapter's accept loop has NO recover (`grep -rn "recover()" pkg/adapter/base.go`
   returns nothing) and runs the handler on a bare goroutine (pkg/adapter/base.go:373).
   An NBD panic would therefore kill the whole DittoFS process, taking every NFS
   and SMB connection with it. Both siblings recover: pkg/adapter/nfs/connection.go:420
   (handleConnectionClose, deferred at :111) and pkg/adapter/smb/connection.go:474.
   Conn.Serve now recovers and logs at Error with debug.Stack(), which needs
   `github.com/marmos91/dittofs/internal/logger` (a PACKAGE, not a package-level
   var — there is none in internal/adapter/nbd) and `runtime/debug`.

4. A TEST THAT WOULD PASS VACUOUSLY (Important). TestIntegration_ReadEvictedButManifestCoveredRange
   said "Write, Flush, evict, then READ". Probed against the real fixture:
   LocalDurable=true RemoteDurable=false RequireDurableCommit=false, Flush returns
   {Finalized: false} and returns EARLY at pkg/block/engine/flush.go:554 without
   draining the remote, so the bytes stay dirty and journal.Evict skips their
   segment (pkg/block/journal/evict.go:36). Measured: SegmentsEvicted=0 after
   Flush alone, 1 after DrainAllUploads (BytesFreed=65676), and the subsequent
   read returned 0xab — a warm local read that never touched the hydrate path the
   test exists to prove. The plan now mandates DrainAllUploads and an explicit
   `if ev.SegmentsEvicted == 0 { t.Fatalf(...) }`, and says not to "fix" it by
   flipping RequireDurableCommit (that changes the contract under test and slows
   every write, per the flush.go:547 comment).

5. AN UNWRITABLE TEST SPEC (Important). The Degraded test was specified for
   pkg/adapter/nbd/healthcheck_test.go (package nbd_test), but the increment site
   is in serve() BETWEEN ServeHandshake and the store resolution — a few
   microseconds on the server's own goroutine, unreachable from outside. Probed
   that the failure path is real: ResolveForWrite succeeds while the share exists
   and fails with `share "/export" not found` after RemoveShare. Split: the
   increment half moves to internal/adapter/nbd/connection_test.go (package nbd)
   with an injected failing StoreResolver; the external test pins only the
   observable contract (a probe with no failures does not invent degradation).

6. WRONG CALL-SITE COUNT (Minor). The rename was said to touch "seven call sites"
   in internal/adapter/nfs/v4/handlers/content_errmap_wiring_test.go. That file
   contains ONE COMMENT mentioning the name and zero calls; it does not even
   compile-depend on it. Real sites: read_payload.go:60, write_payload.go:68,
   :122, :171.

7. pipe(t) REDEFINITION (Minor). connection_test.go is package nbd, the same
   package as handshake_test.go, so the pipe(t) helper is already in scope; the
   plan now says not to redefine it and points at the net.Pipe reasoning.

Also removed: the `func (a *Adapter) Handler() *nbd.Handler` accessor. It was dead
— no test in Tasks 4-8 uses it — and with the Degraded test moved to the internal
package nothing needs it. Rejected: adding it back "just in case", and reaching
into the counter from an external test.

## Task 4 — deferred minor confirmed, NOT fixed here

createNBDAdapter (cmd/dfs/commands/start.go:1024-1053) reads only bind_address,
allow_remote_bind, default_export and max_connections, so Config.Timeouts,
MaxMessageSize, UID and GID carry mapstructure tags nothing honors: an operator
setting adapters.nbd.timeouts.read in the DB row gets the compiled-in defaults.
Confirmed the impact is real rather than cosmetic — Task 4 Step 6 reads
a.config.timeouts() and a.config.UID, so those values reach the handler and are
always the defaults.

NOT fixed in Task 4, and deliberately: the same gap exists for SMB today
(pkg/adapter/smb/config.go has 26 mapstructure tags; createSMBAdapter at
cmd/dfs/commands/start.go:959-978 reads bind_address and signing and nothing
else), so this is a pre-existing shape across adapters rather than something the
NBD adapter introduces. Fixing it inside Task 4 would widen a task that is already
the largest in the plan, and would do it for one adapter only — leaving the repo
inconsistent in a new way. Ledgered for the whole-branch review to triage, with
the note that the honest minimal options are (a) honor the four tags in
createNBDAdapter, or (b) drop the tags so the config surface does not promise what
it cannot deliver. Option (b) is cheaper and more honest; option (a) matches what
an operator will expect from reading the struct.

## Task 4 — implemented, committed `042ead73`, controller verification

`feat(nbd): connection loop and READ handler` — SSH-signed, tree clean, 14 files,
1533 insertions / 56 deletions. `internal/adapter/nbd/connection.go` 334,
`connection_test.go` 771, `handlers_read.go` 88, `integration_test.go` 214,
`pkg/adapter/nbd/adapter.go` +81, `config.go` +10, `healthcheck.go` +19,
`healthcheck_test.go` +31, `internal/adapter/common/*` the Committer widening and
the normalizeBlockStoreError -> NormalizeBlockStoreError rename.

Verified independently of the report: `go test -race` on
./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/... all
ok; `go build ./...` OK; `go vet` clean; `gofmt -l` empty; signature present.
`Conn.Serve` at connection.go:150-167 closes the socket in a deferred
`_ = c.conn.Close()` placed BEFORE the recover defer, so the recover logs first.

## Task 4 — implementer concerns, ruled on

### Deviation 1: Conn.Serve also closes the socket — ACCEPTED

The brief's Step 6 code block omitted the close, but the brief's own prose and the
whole repo require it, and I verified the claim against the siblings rather than
taking it on trust: pkg/adapter/nfs/connection.go:459 and
pkg/adapter/smb/connection.go:502 both close in their close handler, and Task 3's
committed placeholder closed at pkg/adapter/nbd/adapter.go:147 (`defer func() {
_ = c.conn.Close() }()`). Without it the server's end sits in CLOSE_WAIT, because
the base adapter only unregisters the connection and its force-close runs solely
when the drain deadline expires.

Ruling: keep it. Cost if wrong: none — a close the framework also performs is
idempotent on a net.Conn.

### Deviation 2: DISC handled before the draining check — ACCEPTED

The brief's ordering would answer a DISC with ESHUTDOWN while draining. The NBD
spec is unambiguous: "There is no reply to an NBD_CMD_DISC." Replying to it would
be a protocol error visible to any client that disconnects during a restart — and
a restart is precisely the scenario the drain path exists for, so the two would
collide exactly when it matters.

Ruling: keep it. Cost if wrong: none; the branch also validates that offset and
length are zero and answers EINVAL for a malformed DISC, so a client cannot use it
to skip validation.

### Concern 3: Timeouts.Idle is dead — CONFIRMED as a real defect, to fix

Verified, and it is worse than "dead": the plan's own doc for the field makes a
claim the code does not honour. Both siblings honour Idle — NFS at
pkg/adapter/nfs/connection.go:312 (resetIdleTimeout, called from the connection
loop) and SMB at pkg/adapter/smb/connection.go:230 and :456. NBD arms only Read,
at internal/adapter/nbd/connection.go:203, before every header read.

Two facts that decide the shape of the fix:

  - `Idle` cannot be folded into a min(Read, Idle) shortcut, because 0 is a
    sentinel meaning "no idle timeout" (pkg/adapter/nbd/config.go:42).
  - At the defaults Read == Idle == 5m (config.go:107-115), so behaviour is
    currently correct by accident and the gap is invisible until an operator
    separates them.

Ruling (controller, plan-mandated): Idle becomes the deadline for WAITING for the
next request header, and Read becomes the deadline for FINISHING a request already
started (the payload read that Task 5 adds). Rationale: it makes both fields live,
it makes both doc comments true rather than one of them false, and it matches the
split the siblings already use — NFS arms Idle on the connection loop and Read on
the fragment read. It also removes the need for the plan's "Read must be the
shorter of the two" claim, which was self-defeating: if Read is always shorter and
is armed before every read, Idle can never fire.

Cost if wrong: the header read is bounded by Idle instead of Read. At the defaults
those are equal, so the only observable change is for an operator who sets Idle
below Read, and for that operator the new behaviour is the one the field's own doc
promises. Task 5 must arm Read before the WRITE payload read for this to hold.

Test that pins it and fails today: TestConnection_IdleDeadlineClosesASilentConnection
— Idle 200ms, Read 5s, complete the handshake, send nothing, expect serve to return
within ~1s. Today Read=5s holds the connection open, so the test fails.

### Note: the deferred createNBDAdapter mapstructure gap now reaches timeouts()

The gap was already ledgered. This task makes it observable rather than cosmetic:
Step 6 reads a.config.timeouts(), so Timeouts.Read/Write/Idle reach the handler as
compiled-in defaults no matter what the operator writes in the adapter config.
Unchanged ruling: defer to the whole-branch review, because SMB has the identical
gap and fixing it here would leave the repo inconsistent in a new way.

## Task 5 pre-dispatch audit (commit 63998dfd)

Five plan defects found by reading the committed code before dispatching, not by
trusting the plan's description of it. Same pattern as tasks 1-4: the first
implementer run would have been spent on a defect.

Ruling: newWriteTestHandler is a SECOND fixture, not a change to newTestHandler.
  Why: the read-path helper's nil meta is correct for task 4 and 12 of its call
  sites are READ tests. Changing it would make every read test build a runtime.
  Cost if wrong: two fixtures that could drift; the plan says to factor the
  integration fixture into a shared helper if that is cheap.

Ruling: the drain re-arms its own read deadline rather than sharing the request's.
  Why: SetReadDeadline before ReadRequest covers the 28-byte header; the drain is
  where a stalled client parks, and the huge-write test needs a bound on it.
  Cost if wrong: a legitimate slow client whose payload straddles the deadline
  gets a TCP close instead of an error reply -- but it would have hit the same
  deadline mid-payload on the success path anyway.

Ruling: TestConnection_WriteWithNoHoleFlagIsEINVAL is UPDATED, not deleted.
  Why: it pins a real rule (NO_HOLE is not a WRITE flag) that task 6's checklist
  still requires, and the only thing wrong with it is the missing payload.
  Cost if wrong: none; sending the payload the header declares is what a real
  client does.

Ruling: Timeouts.Idle now bounds the header wait (commit 056b547f).
  Why: the field was declared, defaulted, plumbed and never armed, so at the
  defaults (Read == Idle == 5m) the adapter was correct by accident. NFS and SMB
  both arm Idle on the loop and Read on the payload read.
  Cost if wrong: none at the defaults; an operator who sets Idle separately now
  gets what the field name says.

## Task 5 mid-flight ruling (commit 55a4386a)

The implementer stopped before writing code and reported a real contradiction in
the brief: Step 3 and Review Focus item 6 said the reply is written after the
drain, while TestConnection_RejectedHugeWriteDoesNotAllocate asserted ENOSPC
arrives for a client that stalls mid-payload. Both cannot hold.

Ruling: reply FIRST, then drain (Option B of the two the implementer offered).
  Why: the reply is server->client and the payload is client->server, so neither
  orders the other. The drain's obligation -- consume exactly req.Length bytes so
  the next header is read from the right offset -- is unchanged by when the reply
  is written. The plan's ordering sentence asserted a constraint that does not
  exist, so the sentence was the defect, not the test.
  Why not Option A (keep drain-first, drop the assertion): under drain-first a
  client that declared a huge length and stalled gets NO reply at all, only a TCP
  close, which its kernel reports as a transport failure (EIO) rather than the
  ENOSPC it could act on. Option A would have deleted an assertion that was
  correct in order to preserve an ordering claim that was not.
  Cost if wrong: a client that abandons the connection immediately on seeing the
  error leaves the drain to time out; Serve then closes a connection the client
  had already abandoned. No data is at risk and no reply is lost.

Also corrected in the same commit: the NO_HOLE test's failure mode. With
reply-first it reads its reply fine and is left draining a payload that never
comes, rather than hanging on readReplyHeader as the previous text said.

Ledger note: the implementer asked rather than choosing, which is the contract.
This is the second time a brief contradiction was caught by an implementer
stopping (task 3's SetRuntime/Serve test was the first), and both times the
ruling was to fix the plan, not the guard.

## Task 5 (dispatch 2, run b0da839b) — implementer escalation and ruling

- Task 5 implementer asked before writing code: the brief said the rejection
  reply is written AFTER the payload drain, but
  `TestConnection_RejectedHugeWriteDoesNotAllocate` sends `Length: 1<<30`, a
  little payload, then stalls, and asserts ErrENOSPC arrives within
  Timeouts.Read. Drain-then-reply makes that assertion unreachable — a stalled
  client never completes the drain, so no reply is ever written.
- **Ruling: reply first, then drain.** Commit 55a4386a. The spec requires the
  client to send the payload unconditionally ("The client MUST follow the
  request header with *length* number of bytes to be written to the device"),
  so a server that intends to refuse must still consume it. Writing the reply
  first lets a client that is stalling on its own payload learn the outcome
  immediately, and the drain still has to succeed or the connection closes
  (stream position unknown).
  Cost if wrong: a client that pipelines a second command before reading the
  first reply sees the reply earlier than it expected, which is legal
  (replies need not be ordered) — no protocol violation either way.
- The implementer was right to escalate rather than choose; the brief
  contradicted itself and the two options implied different test semantics.

## Task 6 pre-dispatch audit — six defects fixed (a6aace2c)

1. **The metadata punch was missing entirely, and the feature would have
   silently not worked.** `engine.Store.PunchHole` reaps CAS blocks only inside
   `if len(currentBlocks) > 0` (`pkg/block/engine/readwrite.go:308`). The plan
   called it with `nil`, so the reap — `DecrementRefCountAndReapMany` plus
   `ReprojectBlocks` — was skipped and only the zero-overwrite ran: reads return
   zeros while every chunk stays referenced and the manifest keeps listing them.
   A discard that frees nothing is the one thing a discard exists not to do.
   TRIM/WRITE_ZEROES now run the same two-step sequence NFS DEALLOCATE uses
   (`internal/adapter/nfs/v4/handlers/deallocate.go:67`, `:91`):
   `metaSvc.PunchHole` then `blockStore.PunchHole(..., res.PreOpBlocks, ...)`.
2. **The same omission bypassed authorization.** `metadata.Service.PunchHole`
   calls `checkWritePermission` (`pkg/metadata/sparse.go:80`);
   `engine.Store.PunchHole` checks nothing. WRITE goes through `PrepareWrite`,
   which checks. A TRIM straight to the engine would have been the only
   mutating command in the adapter no permission funnel sees — and it destroys
   data. CLAUDE.md invariant 2.
3. **The range clamp underflowed.** `end := min(offset+length, size); length :=
   end - offset` wraps to ~2^64 when `offset > size`. Measured with a Go probe:
   `offset=10000, length=100, size=8192` → `18446744073709549808`. Out-of-range
   is now refused, not clamped.
4. **A design-spec contradiction this exposed.** §4.4 said TRIM/WRITE_ZEROES are
   "clamped to the export end"; §4.6 and §13 said a TRIM past the end is
   EINVAL. The spec's per-command error guidance is the more specific rule and
   wins. Both spec sections corrected in the same commit.
5. **WRITE_ZEROES past the end is ENOSPC, not EINVAL.** The spec gives write
   zeroes its own rule, separate from read and trim.
6. **The read-only fixture comment was wrong.** A read-only export still
   advertises SEND_TRIM (`handshake.go:101` sets it unconditionally and only
   adds READ_ONLY), so the EPERM case is the one the spec names explicitly, not
   a client violating the transmission flags.
   Also: `storeFake` needs a `punches` counter and a `punchErr` field — the
   tests and the Step 7 bite proof both assert on a record the fake never kept;
   and Step 8 no longer runs `make test-unit`, which times out on five unrelated
   packages under `-race` on this machine.

## Task 7 pre-dispatch audit — two defects fixed (d229e29f)

1. **`qemu-nbd` needs `--discard=unmap` or the oracle lies.** It starts with
   `int flags = BDRV_O_RDWR` (`qemu-nbd.c:589`) and `BDRV_O_UNMAP` is a separate
   bit (`include/block/block-common.h:246`) that only `--discard=unmap` sets
   (`bdrv_parse_discard_flags`, `block.c:1154`). `bdrv_co_pdiscard` opens with
   `/* Do nothing if disabled. */ if (!(bs->open_flags & BDRV_O_UNMAP)) { return
   0; }` (`block/io.c:3185`), so a TRIM is dropped and reported as success.
   Without the flag the oracle's discard step diverges on the REFERENCE server's
   configuration and reads as a codec bug — the expensive kind of false
   positive, because the obvious fix is to make DittoFS stop zeroing the range.
   A server that drops a discard is legal ("A server MAY discard length bytes
   starting at offset, but is not required to"), so this is a configuration the
   comparison must pin, not a qemu-nbd defect.
2. **The export-name spelling.** QEMU strips exactly one leading `/` from the
   URI path (`nbd_parse_uri`, `qemu-nbd.c:1563-1569`), so
   `nbd://127.0.0.1:<port>/export/boot` sends `export/boot`. It resolves because
   `RuntimeExporter.Resolve` folds the share half with
   `metadata.NormalizeShareName` (`pkg/metadata/validation.go:176`). Written
   into the script's comments because the volume lists as `/export/boot` and a
   reader would otherwise "fix" the URI to a double slash and get the same
   result for a different reason.
   Also: `-b 127.0.0.1` written explicitly (the oracle holds a scratch file and
   has no business listening off-loopback).

## Task 8 pre-dispatch audit — one defect fixed (fe5b3509)

**`dead_conn_timeout` is not configured the way the brief implied.** The brief
said "the script must configure `dead_conn_timeout` explicitly" without naming
the flag or the path, and the obvious reading is wrong in a way that fails
silently:

1. The flag is `-T <seconds>` / `--dead-timeout <seconds>` (`args.c:76`, handled
   at `:199`). It is **not in `nbd-client`'s man page** — `grep -i dead
   man/nbd-client.8.sgml.in` returns only "deadlock" — so it is undiscoverable
   from `man nbd-client`. The script has to name it.
2. It reaches the kernel **only as `NBD_ATTR_DEAD_CONN_TIMEOUT` over netlink**
   (`nbd-client.c:185-186`). On the ioctl path the option is parsed, stored, and
   **never read**: every other reference to `dead_conn_timeout` in
   `nbd-client.c` is inside the netlink send. With `-nonetlink`/`-L`, or a
   `nbd-client` built without netlink support, the script would print a
   configured window the kernel never received — the same silent failure the
   whole design exists to eliminate, one layer down. The harness must assert the
   mechanism took effect (`check_conn`, `nbd-client.c:394`, already refuses when
   the netlink family will not resolve) rather than fall back.
3. **`-persist` alone already supplies 30 seconds** (`nbd-client.c:188`:
   `else if (cur_client->persist) NLA_PUT_U64(msg, NBD_ATTR_DEAD_CONN_TIMEOUT,
   30); /* Default 30 seconds for persist mode */`). The default happens to sit
   inside the hung-task window; the script still sets `-T` explicitly, because a
   default that happens to be safe is not a configured one.
4. The kernel side is from 2017 (`560bc4b39952ed77cdb0000992e9415b0ee89edb`,
   "nbd: handle dead connections"), so the requirement to check is
   `nbd-client`'s netlink support, not the kernel version.

## Task 5 — WRITE, FUA, FLUSH (commits 6fa295e3, base 7b284dfc)

### The plan defects found before and during the run

1. **`mapError` had no `ErrAccessDenied` arm — a real bug in Task 4's committed
   code.** The metadata permission funnel returns `merrs.ErrAccessDenied`
   (`pkg/metadata/auth_permissions.go:510`, `:539`), not `ErrPermissionDenied`.
   Different codes: `ErrAccessDenied` is POSIX EACCES ("the caller lacks the
   required read/write/execute permission bits", `pkg/metadata/errors/errors.go:22-25`)
   and `ErrPermissionDenied` is EPERM ("requires ownership or root privileges",
   `:29-32`). Task 4's switch handled only the latter, so a write refused by
   POSIX mode bits fell through to the default arm and reached the client as
   `NBD_EIO` — a hardware-failure report for an authorization decision.
   Fixed in `internal/adapter/nbd/connection.go:328`
   (`case merrs.ErrPermissionDenied, merrs.ErrAccessDenied:`). **Bite proof:
   removing the arm → `integration_test.go:301: WRITE refused by file
   permissions reply error = 5, want ErrEPERM (1)`, with the log line
   `[ERROR] nbd: unmapped block store error error=AccessDenied: write permission
   denied`.**
2. **The fixture's identities did not match, and that is what stalled the run.**
   `newIntegrationFixture` creates the volume as UID/GID 1000
   (`integration_test.go:105-107`) while `writeVolumeFixture.newHandler` builds
   the handler as `Identity{UID: 65534, GID: 65534}` (`:62-63`), so every
   happy-path WRITE hit `AccessDenied: write permission denied` from
   `PrepareWrite`. Cause: `ApplyOwnerDefaults` (`pkg/metadata/validation.go:465`,
   called from `pkg/metadata/file_create.go:389`) assigns the *creating*
   identity, and the mode bits are then evaluated against whoever writes.
   Verified directly against a real runtime: create as UID 0 + write as 65534 on
   0644 → `AccessDenied`; create as 65534 + write as 65534 → `PrepareWrite OK`.
   The implementer had independently reached the same wall and had left a
   throwaway `zz_dbg_test.go` probe; it removed the probe before committing.
   `integrationIdentity` and `newIntegrationFixtureAs` were added so the fixture
   can create as the identity the handler serves as, and the mismatch is kept as
   `TestIntegration_WriteDeniedByPermissionIsEPERM` — deliberately, because it is
   the only coverage for the new arm.
3. **The reply-before-drain ruling (commit 55a4386a) was implemented exactly as
   ruled**, and its reasoning is reproduced in the code at
   `internal/adapter/nbd/handlers_write.go` — `rejectWrite` writes the reply,
   re-arms `Timeouts.Read`, then `io.CopyN(io.Discard, conn, int64(req.Length))`.

### Verification performed independently of the report
- 12 write/flush tests listed and run individually under `-v`, all PASS.
- `go test -race ./internal/adapter/nbd/... -count=1` → ok, 7.304s.
- `go build ./...` clean; `go vet` clean; `gofmt -l` empty; `test/no-ai-attribution.sh 55a4386a` → "no AI attribution in 5 commit(s)".
- Signature present (`git cat-file -p HEAD | grep -c "BEGIN SSH SIGNATURE"` → 1).
- **FUA bite proof reproduced by the controller**: moving `h.commit` after the
  reply write → `--- FAIL: TestConnection_WriteWithFUAIsDurableBeforeReply`,
  then restored.
- Review range is exactly the implementer's commit: `git log --oneline 7b284dfc..6fa295e3` → one commit, 5 files, all under `internal/adapter/nbd/`.

### Implementer concerns, assessed
1. **`TestIntegration_FlushRequireDurableWithUnhealthyRemoteIsEIO` needed a
   mechanism change.** The brief's recipe (strict mode + unhealthy remote) cannot
   reach `ErrNotDurableYet`, because `engine.Store.Flush` returns on the
   local-durable fast path at `pkg/block/engine/flush.go:554` before the remote is
   consulted when `!RequireDurableCommit()` — and the journal local tier is
   durable. The committed test additionally calls `f.bs.Local().SetDurable(false)`
   and asserts `!f.bs.LocalDurable()` before proceeding, so it cannot pass
   vacuously. **Accepted**: `SetDurable` is a real method on every local and
   remote store (`pkg/block/journal/admin.go:28`,
   `pkg/block/local/memory/memory.go:407`, `pkg/block/remote/s3/store.go:132`,
   `pkg/block/remote/memory/store.go:234`) and `bs.Local()` is production
   (`pkg/block/engine/engine.go:369`), not a test hook. Intent preserved.
2. **The allocation assertion in `TestConnection_RejectedHugeWriteDoesNotAllocate`
   is weaker than it reads** — `runtime.ReadMemStats` deltas are noisy and the
   `>= declared` threshold is coarse. **Accepted as a deferred minor**: the
   structural guard is what actually holds (`io.CopyN` to `io.Discard` at
   `handlers_write.go:146`, no `make` on the reject path; the only
   `make([]byte, req.Length)` is at `:69`, on the accepted path, after all four
   bounds checks).
3. **`newWriteTestHandler(t, st, exp)` became `writeVolumeFixture.newHandler(t, st)`**
   — an interface deviation from the brief, because the write tests need the
   resolved export and the runtime's metadata service together. **Accepted**: it
   is a test helper and the brief's signature could not have worked — the
   handler needs the export and the service from the same runtime, and a bare
   `(t, st, exp)` function would have to take the runtime too.

   **CORRECTION (controller, after verification):** the claim that
   `newWriteTestHandler(t, st)` was "retained as a thin wrapper" is FALSE. No
   such function exists — `grep -rn newWriteTestHandler internal/` returns
   nothing. The struct and its method are the whole fixture. The plan has been
   corrected to declare the real shape (`writeVolumeFixture` at
   `integration_test.go:48`, `newHandler` at `:69`) rather than the function that
   was never written, so Task 6's brief no longer instructs an implementer to
   call a helper that does not exist.

## Task 6 pre-dispatch audit, round 2 — two more defects found while the Task 5 review ran

Both were found by checking the plan's own Go against what Task 5 actually
shipped, which is the pattern that has now caught a defect before every dispatch
in this project.

1. **`newWriteTestHandler` does not exist.** Task 6's brief told its implementer
   to use "Task 5's `newWriteTestHandler`" for every test expecting
   `Error == 0`. Task 5 built `writeVolumeFixture` with a `newHandler` method
   instead (`internal/adapter/nbd/integration_test.go:48`, `:69`) — correctly,
   since a write test needs the resolved export and the runtime's metadata
   service from the same runtime. Fixed in commit `4e8781a1`: the plan now
   declares the struct and method, and the ledger's false "retained as a thin
   wrapper" claim is corrected in place.

2. **`return mapError(err)` does not compile, and the compiled variant would
   hang the client.** `mapError` returns `NBDError`, a `uint32` with no `Error()`
   method (`internal/adapter/nbd/protocol.go:76`). Two snippets in the plan (Task
   5's write path at plan:2060 and Task 6's punch sequence) returned it from
   `error`-returning handlers. Fixed in commit `ca32c714`: both now go through
   `h.writeReply(conn, req.Cookie, mapError(err), nil)`, which is what the
   shipped `handleWrite` and `handleRead` do (`handlers_write.go:88`,
   `handlers_read.go:84`), with a note that there is deliberately no helper
   converting a store error into a returned error — it would put the reply in two
   places.

**Ruling: correct the plan, not the code, in both cases.** Cost if wrong: none —
these are documentation defects in a plan whose snippets are illustrative; the
shipped code was already right.

**Note for the final review:** `newWriteTestHandler` is now absent from the plan
entirely, so if a later task's brief mentions it, that brief is stale.

## Task 6 pre-dispatch audit, round 3 — the integration assertion was UNWRITABLE AS SPECIFIED

Found by writing a throwaway probe against the real fixture and running it, not
by reading. Both of Task 6's Step 6 integration tests were specified to assert
"via `bs.DataExtents` that the trimmed range is no longer reported as data".
**That assertion cannot pass, on any input.** Measured:

| probe | result |
|---|---|
| 1 MiB written, 256 KiB punched, `bs.DataExtents` | `[[0 1048576]]` — unchanged |
| punch aligned to a real chunk extent, 4 MiB random data | `[[0 4194304]]` — unchanged |
| manifest `file.Blocks` after the same punch | **4 blocks → 3 blocks**, 4194304 → 3095584 bytes referenced |
| read of the punched range | 0 non-zero bytes of 4096 — zeros, as designed |

Two independent reasons, both structural:

1. **The local tier's answer is "has a record here", and the punch writes a
   record.** `engine.PunchHole` zero-overwrites the range through the local
   append log (`pkg/block/engine/readwrite.go:281`) — that is what guarantees
   reads return zeros regardless of rollup state, and it is why the comment says
   the zero chunks dedup to one CAS object. The side effect is that the local
   tier now has an interval covering the punched range, so it reports data.
2. **`DataExtents` over-reports on purpose.** Its own doc: *"The map may
   over-report data but must never under-report it"* (`pkg/block/engine/dataextents.go:41`),
   because NFSv4.2 SEEK/READ_PLUS callers must never skip real data. A hole
   reported where a punch happened is exactly the over-report the polarity
   permits — a client copying zeros is safe.

**What IS observable, and what the tests must assert instead:** the manifest.
`file.Blocks` drops from 4 to 3 and the referenced byte count falls by the
punched block's size. That is the reclaim, and it is what `metadata.Service.PunchHole`
produces by pruning `file.Blocks` and persisting the new manifest.

**Ruling: rewrite both integration assertions to read `file.Blocks` through
`rt.GetMetadataService().GetFile(ctx, exp.Handle)`, before and after, and assert
the count and byte total fall. Do NOT assert `DataExtents` shrinks** — that
assertion is unsatisfiable and would have burned an implementer run and then a
review round. Cost if wrong: the test asserts manifest state rather than the
extent map, which is a slightly more internal view; it is also the only view in
which the reclaim is visible at all.

**Lesson, and it generalises:** an assertion about a primitive whose documented
contract is "over-report" cannot be written as "stops reporting". Check the
polarity of the primitive before specifying the assertion, not after the test
fails.

## Forward-reference scan, re-run after the Task 5/6 corrections — clean

A Python scan over `## Task N:` sections, collecting every declared symbol from
each task's Go fences and matching bare (unqualified) references against later
tasks' declarations, now reports **zero real forward references**. It reports 14
hits, all false positives of one kind: **Task 4 Step 6 and Step 7 deliberately
re-declare symbols Task 3 introduced** — `Adapter`, `Serve`, `SetRuntime`,
`Healthcheck` — because that step replaces Task 3's minimal bodies with the wired
versions. The scan sees the same names in two tasks and calls the earlier one a
forward reference; the plan documents the replacement in both places, so it is the
intended shape. Same false-positive class as `commit`, which appears in bash
fences as `git ... commit -S`.

**Nothing to fix. Recorded so the next scan's output is not mistaken for a
regression.**

## Task 6 pre-dispatch audit, round 4 — `Timeouts.Idle` is STILL dead, and the plan/code now diverge

Found by reading the committed loop, then proved with a throwaway probe. This is
the second time this field has come up; the first time (commit `056b547f`) I
corrected the PLAN and ruled that Task 5 must arm `Read` before the payload read.
That ruling stands, but the plan-side fix never reached the code, because Task 4
was already committed when it landed.

**Measured, not reasoned.** Probe: `Timeouts{Read: 5s, Idle: 200ms}`, complete the
handshake, send nothing.

```
serve still running after 1.5s: Idle is NOT armed (Read=5s is holding it)
```

The committed loop at `internal/adapter/nbd/connection.go:203` arms
`SetReadDeadline(time.Now().Add(h.timeouts.Read))` before every `ReadRequest`,
with no `Idle` arm anywhere. The plan mandates the `budget := h.timeouts.Read; if
h.timeouts.Idle > 0 { budget = h.timeouts.Idle }` block at plan:1709-1715, and
`TestConnection_PartialHeaderTimesOut` (`connection_test.go:696`) sets
`Idle: 5s, Read: 200ms` — the one arrangement in which the two are
indistinguishable, so no committed test can fail on the missing arm.

**Ruling: land the fix in Task 6, not Task 4.** Task 4 is committed and reviewed;
re-opening it for a two-line change costs a fresh implementer plus a scoped
re-review, while Task 6 already touches `connection.go` (it adds the dispatch arms)
and already runs the package tests. Cost if wrong: Task 6 carries two extra lines
and one extra test, versus shipping an operator-visible field that does nothing.

**The test that must land with it:** `TestConnection_IdleDeadlineClosesASilentConnection`
— `Timeouts{Read: 5 * time.Second, Idle: 200 * time.Millisecond}`, complete the
handshake, send nothing, and require `serve` to return within ~1s. It fails today
(the probe above is the proof) and passes with the block. Note it must NOT be
written with `Idle > Read`: that arrangement passes either way.

**Also recorded:** the plan and the code now disagree about this one block, so the
whole-branch review will see a plan-mandated change that is not in Task 4's diff.
That is expected and explained here — do not read it as a missing implementation.

## Task 6 pre-dispatch audit, round 3 — the Timeouts.Idle arm (commit a533e7f5)

**A plan edit landed AFTER Task 4 was already committed, so its code never got
the change.** `056b547f` (2026-10-12) added the Idle/Read split to the plan's Task
4 Step 3, but Task 4's commit `042ead73` is from 2026-10-09 — the implementer had
already written the loop, and nothing re-opened it. The plan then described code
that does not exist, which is the failure mode that makes a plan worse than
useless: a later reader trusts it.

**What the code does** (`internal/adapter/nbd/connection.go:203`): one arm, from
`h.timeouts.Read`, before every header read. `h.timeouts.Idle` is declared,
defaulted, validated, plumbed into the handler and **never read** — grep finds it
only in `config.go` and the test fixtures. So `Timeouts.Idle`'s own doc
(`pkg/adapter/nbd/config.go:42`, *"the maximum duration a connection may remain
idle between requests"*) is false, and at the defaults `Read == Idle == 5m` so
nothing observable is wrong — the field only matters to an operator who
separates them, which is exactly the operator the doc is written for.

**Measured, not reasoned:** with `Timeouts{Read: 5s, Idle: 200ms}` and a silent
post-handshake connection, `serve` was still running after 1.5s. It should have
returned in ~200ms.

**Two independent gaps, both now in Task 6 Step 5:**
1. The header wait is armed from `Read`, not from `Idle`.
2. **`Read` bounds nothing either.** `handleWrite` reads the payload through
   `io.ReadFull` (`internal/adapter/nbd/handlers_write.go:69`) under whatever
   deadline the header read left behind, so no arm covers the payload read. After
   fix 1, the payload would inherit the *Idle* budget — the two timers would be
   swapped rather than fixed.

**Why Task 6 and not a Task 4 fixup:** Task 6 already modifies `connection.go`
and `handlers_write.go`, both halves land in one commit, and fix 2 is in Task 6's
own `handleWrite`. Amending Task 4 would rewrite a reviewed, approved commit for
a defect nothing else depends on.

**The test must separate the timers or it passes either way.** The committed
`TestConnection_PartialHeaderTimesOut` (`connection_test.go:696`) uses
`Idle 5s / Read 200ms` — the wrong way round to catch this. The new
`TestConnection_IdleDeadlineClosesASilentConnection` uses `Read 5s / Idle 200ms`
and asserts `serve` returns within ~1s.

**Ruling: the plan is right and the code is behind it; land the code, keep the
plan.** Cost if wrong: the header wait is bounded by Idle instead of Read, which
at the defaults is the same number, and for an operator who sets Idle below Read
the new behaviour is what the field's own documentation already promises.

**Lesson for the remaining tasks:** any plan edit that names a line of already-
committed code must carry a step that re-opens that code, or it describes a
program that does not exist. `056b547f` did not, and this audit is the cost.

## Task 5 — REVIEW APPROVED (reviewer run 779e8e47-1535-4695-870f-28a05fba84c4)

**Verdict: Approved. No Critical, no Important. Two Minors, both report-only.**

All eight checks met with file:line evidence:
1. Check order + subtraction overflow — `internal/adapter/nbd/handlers_write.go:33` flags → `:38` read-only → `:46` zero-length (no drain) → `:52` `req.Offset > math.MaxUint64-uint64(req.Length)` (subtraction) → `:60` past-end ENOSPC → `:68` payload → `:86` `PrepareWrite` → `:93` `bs.WriteAt` → `:100` `CommitWrite` → `:107` FUA commit → `:113` reply.
2. Desync guard — `rejectWrite` replies at `:133` then `io.CopyN(io.Discard, conn, int64(req.Length))` at `:144`; zero-length path (`:46-48`) never drains; every non-zero-length rejection routes through `rejectWrite`.
3. FUA ordering — `:107-111` commit before `:113` reply.
4. `commit` delegates — `handlers_control.go:49` `return common.CommitBlockStore(ctx, bs, exp.PayloadID)`; grep for `RequireDurableCommit|LocalDurable|RemoteDurable|ErrNotDurableYet` in non-test `internal/adapter/nbd/` returns only test and comment hits, so the durability rule appears exactly once.
5. `mapError` — `connection.go:301` normalizes first; `:333` `case merrs.ErrPermissionDenied, merrs.ErrAccessDenied:`; only EIO/EPERM/ENOSPC/ESHUTDOWN/0 are returned; `protocol.go:80-84` defines no EOVERFLOW/ENOTSUP.
6. FUA test mechanism — parks the server inside `storeFake.Flush` via `st.onFlush`, then reads the reply with a 250 ms deadline while parked and fails if any header arrives. Moving `commit` after the reply makes the reply readable while parked, so it fails on ordering, not on `st.flushes`.
7. Stream-sync test — sends the full 4096-byte payload for an out-of-range WRITE, then a valid READ at offset 0, asserting `Error == 0` and byte equality with the pre-seeded `0x5a`.
8. Happy-path WRITEs use a real metadata service — `writeVolumeFixture.newHandler` (`integration_test.go:69-73`) passes `f.rt.GetMetadataService()`, and the fixture creates the volume as `integrationIdentity{65534,65534}` matching the handler; the mismatch lives only in `TestIntegration_WriteDeniedByPermissionIsEPERM`.

**Minors deferred to the whole-branch review:** (a) `TestConnection_RejectedHugeWriteDoesNotAllocate`'s `runtime.MemStats.TotalAlloc` delta is process-global and would be flaky under `t.Parallel()` — no current test parallelizes, so latent only; (b) `handlers_write.go:69`'s `io.LimitReader` is redundant with the exact-size slice passed to `io.ReadFull` — harmless.

**Controller note on the review process, worth fixing for Tasks 6-8:** the `reviewer` agent has **no write or edit tool** (its allowlist is read/grep/find/ls/watchdog_diff/contact_supervisor). Two reviewers on this task therefore could not write their file: the first burned its whole 30-minute budget and returned nothing usable; the second produced a complete verdict but had to return it inline. **From Task 6 on, dispatch reviewers to return the review in their reply and let the controller persist it** — or state explicitly that the reviewer must reply with the full text rather than a file path. The file-writing instruction in `task-reviewer-prompt.md` assumes a tool the agent does not have.

## Task 7 + Task 8 pre-dispatch audits (commits 3ade3fe6, eabfa63c)

### Task 7 — the oracle had no server to point at, and no honest way to get one

The brief's `**Interfaces:**` said "Consumes: the running DittoFS server with an NBD adapter" and never said how to start one. There is no such thing: `pkg/config/config.go:37` says *"Dynamic configuration (users, groups, shares, stores, adapters) is managed through the REST API and stored in the control plane database"*, and `cmd/dfs/commands/start.go:967` sets `Enabled: true` unconditionally in the adapter factory, so there is no "off" a config file could express. A live run needs `dfs start` plus four `dfsctl` calls (`store metadata add`, `store block add`, `share create`, `adapter enable nbd`) — `test/scenarios/setup.sh:73-81` already does exactly this for smb and nfs.

**But a live server still cannot serve this oracle**, for two independent reasons:
1. **No command creates a volume.** `RuntimeExporter.Resolve` (`internal/adapter/nbd/volume.go`) looks up the share root, then `volumes/`, then the volume name; `grep -rn "volume" cmd/dfsctl/` returns nothing; `internal/adapter/nbd/volume.go:21` records that *"the operator, not the protocol, decides who owns the volumes"*, so the operator creates `volumes/<name>` through the share's own NFS/SMB mount.
2. **The volume is a file inside a share**, so a byte-level oracle also needs an fs-backed local tier — a second piece of production configuration, not a test fixture.
A script that stood both up would be testing share provisioning, not the codec, and would be unusable in CI.

**RULING — split the oracle in two.** Cost if wrong: Task 7 produces a Go test instead of a pure shell script, and a reviewer expecting only `test/nbd/*.sh` sees one extra file.
- **Tier A** — `internal/adapter/nbd/oracle_test.go` (external `package nbd_test`): serve the real `Handler` on a real TCP listener, drive it with the real `qemu-io`, diff against `qemu-nbd`. The whole codec is reachable without the control plane because `Handler` is built from an `Exporter`, a `StoreResolver`, a `*metadata.Service`, an `Identity` and `Timeouts` — all test-suppliable. Precedent that no new machinery is needed: `pkg/adapter/nbd/adapter_test.go:88` already serves a real listener, `internal/adapter/nbd/handshake_test.go:50` already does a real TCP handshake.
- **Tier B** — the same script drives a deployed server when `DITTOFS_NBD_PORT` is set, and skips loudly without it. The script **prints which tier ran**; a green run that does not say what it checked means nothing.

**Also pinned, all discovered by reading the code:**
- `test/required-tests.json` is deliberately **NOT** touched: `test/required-tests.sh` runs `go test` for every name in the manifest, so listing a QEMU-dependent test there would fail the required set on every machine without QEMU. The manifest's purpose is the opposite — names whose removal must be loud.
- The Tier A test must **skip, not fail**, without QEMU (`t.Skip` naming the missing binary), and `run-oracle.sh` treats that skip as a failure. That split is what keeps `go test ./...` green without QEMU while the script still refuses to report a check it did not run.
- **`testExportName`, `pipe`, `newTestHandler`, `writeVolumeFixture` are all in `package nbd` test files** (`internal/adapter/nbd/connection_test.go:129` etc.), so an external `package nbd_test` oracle test cannot see any of them and defines its own.
- Use a **fake `Exporter`**, not `nbd.NewRuntimeExporter`: export resolution is `volume_test.go`'s subject, and routing through the runtime would drag provisioning back into the test the split exists to keep it out of. `nbd.Export`'s fields are exported (`internal/adapter/nbd/handshake.go:82`) so a literal suffices.
- The metadata service must be **real** (`rt.GetMetadataService()`): `handleTrim`/`handleWriteZeroes` call `h.meta.PunchHole`, which panics on nil — the same reason Tasks 5 and 6 needed `writeVolumeFixture`.
- Accept-loop entry point is `nbd.Conn.Serve(ctx)` (`internal/adapter/nbd/connection.go:150`), so each accepted conn becomes `nbd.NewConn(h, c)`.
- **QEMU is not installed on this machine** (`command -v qemu-nbd qemu-img qemu-io` → nothing; `qemu-system-aarch64` absent too), so Tasks 7 and 8 both need `brew install qemu` before they can run. The plan now says so at Task 7 Step 4 and instructs the implementer to report unverified steps rather than write a script it never ran.

### Task 8 — the launch line was wrong, in both documents

`-netdev user,id=n0,hostfwd=tcp::10809-:10809` was carried over from the spec's §9.5 into the plan's reference launch. **`hostfwd` forwards a HOST port INTO the guest** (`hostfwd=[tcp]:[hostaddr]:hostport-[guestaddr]:guestport`), which is the opposite direction from what the topology needs, and on a host where DittoFS is already listening on 10809 it fails to bind at startup. The spec's own prose described it wrongly too (*"`hostfwd` maps the guest's loopback to the host's NBD port"*).

Nothing is needed: QEMU's user-net gateway `10.0.2.2` is the host as seen from inside the guest, and a **guest-initiated outbound** connection to it crosses SLIRP with no rule — the same path the guest uses to reach the outside world. `nbd-client 10.0.2.2 10809 /dev/nbd0` is exactly that direction. Fixed in the plan's topology bullets, the plan's reference launch, the spec's §9.5 launch, the spec's §9.5 prose, and added to the Verification Checklist.

### Task 8 — `dead_conn_timeout` (recorded earlier, still in force)
The flag is `-T <seconds>` / `--dead-timeout <seconds>` (`args.c:76`, handled at `:199`) and is **absent from `nbd-client`'s man page** (`grep -i dead man/nbd-client.8.sgml.in` finds only "deadlock"). It reaches the kernel **only as `NBD_ATTR_DEAD_CONN_TIMEOUT` over netlink** (`nbd-client.c:185-186`) — on the ioctl path the option is parsed, stored and never read, so `-nonetlink`/`-L` prints a window the kernel never received. `-persist` alone already supplies 30 seconds (`nbd-client.c:188`). The harness must fail when netlink is unavailable rather than fall back; `check_conn` (`nbd-client.c:394`) is the check.

## Task 6 — implemented (7473656d), and one measured gap fixed in the plan (52138d7d)

**Commit `7473656d`** `feat(nbd): TRIM and WRITE_ZEROES via PunchHole`, SSH-signed, tree clean, 5 files +687/-16: `internal/adapter/nbd/connection.go` +49, `connection_test.go` +360, `handlers_write.go` +147, `integration_test.go` +136, `pkg/adapter/nbd/config.go` +11.

**Verified by the controller, not taken on trust:**
- All 12 TRIM/WRITE_ZEROES/Idle tests PASS individually: `TestConnection_TrimIsEPERMOnReadOnlyExport`, `TrimThenReadIsZeros`, `TrimPastEndIsEINVAL`, `TrimOffsetPastEndIsEINVAL`, `WriteZeroesPastEndIsENOSPC`, `WriteZeroesWithNoHoleIsAccepted`, `TrimZeroLengthIsEINVAL`, `WriteZeroesZeroLengthIsEINVAL`, `WriteZeroesThenReadIsZeros`, `WriteZeroesOnReadOnlyExportIsEPERM`, `IdleDeadlineClosesASilentConnection`, `PartialHeaderTimesOut`.
- Both commands are one-liners delegating to a shared `handleDiscard(ctx, bs, exp, req, conn, allowed CommandFlags, outOfRange NBDError)`: `handleTrim` passes `FlagFUA, ErrEINVAL`; `handleWriteZeroes` passes `FlagFUA|FlagNoHole, ErrENOSPC`. The per-command out-of-range code is a spec requirement, not an inconsistency.
- `handleDiscard`'s check order: flags → `exp.ReadOnly` (EPERM, before length and range, so a read-only export refuses the mutation wherever it lands) → `Length == 0` (EINVAL, **no drain** — neither command carries a payload, so draining would consume the next header) → subtraction overflow guard (`req.Offset > math.MaxUint64-uint64(req.Length)` → EIO) → `req.Offset >= exp.Size` → outOfRange → `req.Offset+Length > exp.Size` → outOfRange. Refused, never clamped, with the underflow arithmetic spelled out in the comment.
- The metadata punch is present with its rationale in code: `h.meta.PunchHole(authCtx, exp.Handle, ...)` then `bs.PunchHole(ctx, string(res.PayloadID), res.PreOpBlocks, ...)`, with the comment naming both reasons — it is the only authorization check on the path (`metadata.Service.PunchHole` calls `checkWritePermission`; `engine.Store.PunchHole` checks nothing) and without it `engine.Store.PunchHole`'s reap is skipped entirely (guarded on `len(currentBlocks) > 0`), so the command would report success and free nothing.
- FUA commits before the reply, same as WRITE.
- Bite proof reproduced by the implementer: zero-buffer `WriteAt` → `connection_test.go:1376: the store saw 0 punches, want 1`; `integration_test.go:535: manifest has 1 blocks after WRITE_ZEROES, want fewer than the 1 before it`; and the `writes == 0` half bites independently (`the store saw 1 writes, want 0`).

### MEASURED GAP — the payload `Read` arm had no test, and the plan's own step never named one

Plan step 5 mandates two arms: `Idle` bounds the header wait, and `Read` is re-armed before `handleWrite`'s payload read. The only test the step names (`TestConnection_IdleDeadlineClosesASilentConnection`) pins the **header** arm — a different arm in a different function.

**Probe:** delete `conn.SetReadDeadline(time.Now().Add(h.timeouts.Read))` from `handleWrite` (the arm before `io.ReadFull`) → `go test ./internal/adapter/nbd/ -count=1` → `ok ... 4.725s`. **The whole package is green with half the split missing.**

`TestConnection_WriteTruncatedPayloadIsAnError` does not catch it for two independent reasons: (1) it uses `testTimeouts()` = `Read 5s / Idle 5s`, so either budget bounds the read and the arm is unobservable; (2) it **closes the client**, so the read returns on EOF and no deadline is ever consulted.

**RULING:** add `TestConnection_PayloadReadIsBoundedByRead` to the plan with the inverse fixture (`Read: 200ms, Idle: 5s`) and a client that stays silent **without closing**, so only the deadline can end the read; with the arm removed the payload read inherits the header's 5s Idle budget and serve is still running at 1s. Cost if wrong: one extra test in the package. Commit `52138d7d`, plus a Verification Checklist row. This is a plan defect, not an implementer defect — the brief could not have produced this test because the plan never asked for it.

### Implementer concerns assessed

1. **`TestConnection_PartialHeaderTimesOut`'s fixture inverted** from `Read 200ms / Idle 5s` to `Read 5s / Idle 200ms`. The implementer's reasoning: a partial header is a *silent wait*, which the split assigns to `Idle`, so the committed fixture had been passing for the wrong reason (it passed because the old code armed `Read` before every header read). **Accepted pending the reviewer's independent judgement** — it is a change to already-committed code, and the question is whether the old fixture was testing something real. It was not: under the old code `Read` bounded the header wait, so the test was asserting the *pre-split* semantics. The inverted fixture is the same scenario under the correct timer.
2. **Two doc comments corrected beyond the brief** (`internal/adapter/nbd/connection.go`'s `Timeouts` doc, `pkg/adapter/nbd/config.go`'s `Read` doc). Verified: the old `Timeouts` doc said *"Read bounds reading one request header or payload. Idle bounds a connection that has completed its handshake but sends nothing, so Read must be the shorter of the two"* — which was self-defeating (if Read is always shorter and armed before every read, Idle can never fire) and is now false under the split. Both new texts match the code. **Accepted.**
3. **`bs.PunchHole` called with plain `ctx`, not `common.WithRequestDeadline(ctx)`** — verified this matches its sibling: `internal/adapter/nfs/v4/handlers/deallocate.go:91` also passes `ctx.Context` to `blockStore.PunchHole`. **Accepted.**
4. **`res.PayloadID` used unguarded** where NFS guards it (`if res.PayloadID != "" && length > 0 && offset < res.File.Size`). Checked: `metadata.Service.PunchHole` sets `res.PayloadID = file.PayloadID` (`pkg/metadata/sparse.go:76`), `buildPayloadID` (`pkg/metadata/file_create.go`) is `strings.TrimPrefix(shareName, "/") + "/" + id.String()` and never returns empty, and the three `PayloadID = ""` sites are all *victim/removed* copies (`pkg/metadata/file_modify.go:1594` rename clobber with `clobberedNlink > 0`, `pkg/metadata/file_recycle.go:143`, `pkg/metadata/file_remove.go:220`) — none of which is a live file a handle can name. NFS's guard is also redundant on its own path for the same reason. **Accepted as a deferred minor**: unreachable today, and an empty `PayloadID` would reach `engine.PunchHole` as a lookup miss rather than corruption.

**Task 6 review dispatched** (reviewer `7c72db34-66c9-48c9-80fe-0b3a9a376bc4`, range `eabfa63c..7473656d` — the parent is `eabfa63c`, so the range isolates the implementer's commit and excludes my three doc commits). The dispatch instructed the reviewer to return the review inline rather than writing a file, and explicitly asked it to judge the payload-arm gap I measured.

## Task 6 review (7c72db34) — verdict "Needs fixes"; rulings

Reviewer agent `reviewer`, model `cubbit/vllm/mimir`, run `7c72db34-66c9-48c9-80fe-0b3a9a376bc4`,
range `eabfa63c..7473656d`. Review persisted by the controller to
`.superpowers/sdd/2026-10-08-dittofs-nbd-block-protocol/task-6-review.md` (the `reviewer`
agent has no write tool — see the process note at ledger:1600). Verdict: 13/13 brief
requirements satisfied, 9/10 priority checks satisfied, 15/15 named tests present, one
Important finding. Reviewer confirmed the metadata-punch sequencing, the no-clamp range
refusal, the per-command out-of-range codes, the zero-length uniformity, and that
`TestConnection_PartialHeaderTimesOut`'s inversion is correct rather than a weakened pin.

**Important — `TestConnection_PayloadReadIsBoundedByRead` missing.**
`Ruling: this is the controller's own mid-flight plan edit, not a brief-vs-plan gap, and the test is in scope for the fix round.`
Verified by timestamp rather than accepted: the plan gained the test at commit `52138d7d`
(2026-10-12 12:19:30) while `task-6-brief.md` was extracted at 11:25:08, so the brief
could not have carried it. The reviewer's framing ("a brief-vs-plan gap") and mine (in the
prior block) were both wrong in the same way — the requirement was committed after dispatch.
`grep PayloadReadIsBoundedByRead task-6-brief.md` → no mention; plan:2463 and plan:2873 → both present.
The reviewer's corollary that "no `Timeouts` literal in the package has `Read < Idle`" is
**factually wrong**: `connection_test.go:953` is `Timeouts{Read: time.Second, Idle: 5 * time.Second}`.
That literal belongs to `TestConnection_RejectedHugeWriteDoesNotAllocate`, which exercises the
**drain** arm, not the payload arm — so the conclusion stands (the payload arm is unpinned) for a
different reason than the reviewer gave. Cost if wrong: none; the plan mandates the test either way.
Consequence for this round: the test lands in `connection_test.go`, which cannot reach
`integration_test.go`'s `newWriteTestHandler`.
`Ruling: re-exporting that helper across the two files in the same package is IN SCOPE.`
Cost if wrong: two lines of package-private test plumbing. Rationale: the plan mandates the test
beside the other connection tests, and the alternative is a second copy of a 6-line fixture helper.

**Minor — FUA on TRIM/WRITE_ZEROES implemented but untested.**
`Ruling: NOT in scope; deferred to the whole-branch review.`
Cost if wrong: the FUA-before-reply ordering on the two discard commands stays pinned only by
reading the code, until the whole-branch review or Task 7's oracle. Rationale: the plan's
Verification Checklist (plan:2855-2885) has **no** FUA-on-discard row — checked, not assumed.
Adding a test for every untested branch makes a task unbounded.

**Minor — `res.PayloadID` used unguarded at `handlers_write.go:248`.**
`Ruling: deliberate omission; leave the code as it is.`
Cost if wrong: a cross-protocol unlink race while an NBD connection is open could call
`bs.PunchHole` with `""` instead of refusing — low likelihood, no crash, and NBD itself has no
unlink. Rationale: `Export.PayloadID` is resolved once at handshake from a regular file and
`buildPayloadID` (`pkg/metadata/file_helpers.go:23`) never returns empty, so the guard is
unreachable on NBD's own paths. Also ruled: do NOT add a comment claiming it is unreachable —
a comment asserting an invariant nobody checks is worse than the unguarded call.

**Minor — bare `ctx` on `bs.PunchHole`.**
`Ruling: no change, and the reviewer's framing is wrong.`
Verified: NFS DEALLOCATE passes the bare context too —
`internal/adapter/nfs/v4/handlers/deallocate.go:93` is
`blockStore.PunchHole(ctx.Context, string(res.PayloadID), res.PreOpBlocks, offset, punchLen(offset, length, res.File.Size))`.
So `handleDiscard` does not "diverge from every sibling store call"; it matches the only sibling
that punches. Cost if wrong: an un-deadlined punch on a huge export, bounded by the engine's own
internal budgets. Rationale for not "fixing" it: changing it would make NBD the outlier.

**Minors IN SCOPE for the fix round:** the stale failure message at `connection_test.go:738`
(names a "read timeout" after the fixture was inverted to `Idle: 200ms`), and a test for the
engine-punch failure arm — `Ruling: ADD the test rather than drop the dead `punchErr` field.`
Cost if wrong: ~10 lines of test. Rationale: it pins a real branch (`handlers_write.go:248-250`,
reachable when the local store is closed), and deleting working test surface to satisfy a
coverage complaint is the wrong direction.

**Fix dispatched as round 1 by resuming the original implementer** (`e1b58c87-a2db-452a-846c-76ed9854da00`).
The dispatch carried an explicit do-not-touch list for (A) FUA-on-discard, (B) the `PayloadID`
guard, and (C) the bare `ctx`, so the implementer would not "helpfully" fix the deferred items.

## Escalation raised to the user (2026-10-12) — QEMU absent, and Task 8's recorded run

Two decisions the SDD run must not take on the user's behalf. Both were surfaced to the
user rather than ruled on; the run continues on everything else meanwhile.

**Escalation 1 — QEMU is not installed on this host, so Task 7's oracle cannot run here.**
Verified: `which qemu-nbd qemu-io qemu-img` → nothing; no `/opt/homebrew/bin/qemu*`; no
`/Applications/*QEMU*`, no UTM. `brew info qemu` → stable 11.1.2, **bottled**, 15 required
deps (capstone, dtc, glib, gnutls, jpeg-turbo, libpng, libslirp, libssh, libusb, lzo,
ncurses, pixman, snappy, vde, zstd). Disk: 94 GiB available. Docker is installed but its
daemon is not running (`failed to connect to the docker API at
unix://<docker.sock>`).
This matters because the plan makes Tier A's evidence the *point* of Task 7: the checklist
row is `internal/adapter/nbd/oracle_test.go` skips (not fails) without QEMU, **and
`run-oracle.sh` treats that skip as a failure** (plan:2881), so Task 7's `run-oracle.sh`
cannot exit 0 on this machine today.
The install is reversible and outside the worktree, which is exactly why it is an escalation
and not a Ruling.

**Escalation 2 — Task 8's Step 6 ("Record a run on a Linux host") is unsatisfiable here.**
Task 8's own brief opens: *"This task cannot run on the development machine (macOS arm64; it
needs a Linux host with KVM, root, and `nbd-client` in the guest initramfs)."* Steps 1-5 and 7
(write both scripts, extend the README, `bash -n`, prove each refuses cleanly without
prerequisites, commit) are all satisfiable on this machine. Step 6 is not, and it is the step
the brief calls *"the point at which the design is falsified or confirmed"*, citing spec §10:
*"Phase 1 is the first moment the design is falsifiable end-to-end, and it is the right place
to stop and reassess."*
So Task 8 can be **written and committed** here but cannot be **verified** here. Ruling pending
on whether that counts as Task 8 complete, or whether the run waits for a Linux host.

**Verified while auditing Task 7 (all clean, no defects):** `go list -deps
./internal/adapter/testruntime` contains zero entries for either `dittofs/internal/adapter/nbd`
or `dittofs/pkg/adapter`, so `package nbd_test` importing it creates no cycle — the plan's
external-test-package requirement holds. Every exported symbol `oracle_test.go` needs exists
with the planned signature: `NewHandler` (`connection.go:93`), `NewConn` (`:140`),
`Conn.Serve` (`:153`), `Timeouts` (`:29`), `BlockStore` (`:42`), `Export` (`handshake.go:82`),
`Exporter` (`:109`), `Identity` (`volume.go:22`). The brief's two cited listener precedents
both exist: `freePort` at `pkg/adapter/nbd/adapter_test.go:88` and `pipe` at
`internal/adapter/nbd/handshake_test.go:50`.
The repo's skip-guard convention to follow is `test/integration/ad-dc/netlogon_smbclient_test.go:42-47`
(`exec.LookPath` then `t.Skip` naming the missing binary), matching the plan's Step 1.
`test/required-tests.json` is Go-test-name only and its own comment says the list is
*"deliberately short… for gates whose removal is silent and expensive"* — which is why the
plan excludes the QEMU-dependent oracle from it. Confirmed correct, not an omission.

## Task 6 fix round 1 — implemented (ac29d710), scoped re-review dispatched

**Commit `ac29d710`** `test(nbd): bound the payload read and the punch failure arm` — SSH-signed
(`git cat-file -p HEAD | grep -c "BEGIN SSH SIGNATURE"` → 1), tree clean, **1 file, +56/-1**
(`internal/adapter/nbd/connection_test.go` only). Base was `7473656d`.
Verified independently by the controller, not taken from the report: `go build ./...` clean;
`go test -race ./internal/adapter/nbd/... ./pkg/adapter/nbd/... ./internal/adapter/common/... -count=1`
→ all ok (`internal/adapter/nbd` 10.070s, `pkg/adapter/nbd` 5.130s, `internal/adapter/common` 8.349s);
`gofmt -l` empty; **75 passing tests** in the nbd package.
The three in-scope items landed:
1. `TestConnection_PayloadReadIsBoundedByRead` at `connection_test.go:1023` — `Timeouts{Read: 200ms, Write: 5s, Idle: 5s}`, WRITE header declaring 4096 with 100 payload bytes, client left **open**, asserts serve returns non-nil within 1s and `st.writes == 0`. Written against `newTestHandler`, so the helper re-export ruled in scope was NOT needed.
2. Stale message at `connection_test.go:739` now reads `"serve did not return within 2s of a 200ms idle timeout: Timeouts.Idle is not armed"` (was "read timeout").
3. `TestConnection_TrimPunchFailureIsEIO` at `connection_test.go:1346` — `st.punchErr = errors.New("boom")`, asserts reply `ErrEIO` and `st.punches == 1`, so it pins the failure arm rather than passing because the punch was skipped.
**Bite proof for the Important finding, both halves:** with the arm at `handlers_write.go:70` removed the package reports **exactly one** failure — `connection_test.go:1042: serve did not return within 1s of a 200ms read timeout: the payload read is not bounded by Timeouts.Read` — and restoring it returns `ok ... 4.903s`. The arm was confirmed present in the committed code at `handlers_write.go:70` (`conn.SetReadDeadline(time.Now().Add(h.timeouts.Read))`, with the doc comment explaining that the header read consumed the Idle budget).
Do-not-touch list was respected: no FUA-on-discard test, no `res.PayloadID` guard, no `WithRequestDeadline` around `bs.PunchHole`.

**Scoped re-review dispatched** — reviewer agent `reviewer`, model `cubbit/vllm/mimir`, async run
`7ef93c2a-438d-4208-85b3-ff8267e36892`. Review package
`review-7473656d..ac29d710.diff` (207 lines, 13118 bytes, **2 commits** — it includes the
controller's own `52138d7d` plan commit, which is correct: that commit is what mandated the
missing test, so the reviewer needs it to judge whether the fix answers the requirement).
The dispatch carried the three sent findings verbatim, an explicit statement that findings 2-4
were ruled deferred/wrong and must not be verdicted as unaddressed, the factual correction about
`connection_test.go:953` so the reviewer does not inherit the original review's wrong claim, and
the instruction to verify that the new test genuinely cannot pass without the arm.
The reviewer has no write tool, so the dispatch requires the review as reply text for the
controller to persist (the standing process fix recorded at ledger:1600).

## Escalation 1 addendum — searched for a QEMU-free oracle, found none

Before asking the user to install 15 dependencies, I checked every lighter path to an
independent NBD oracle. All dead ends:
- `brew info nbd` → *"No available formula with the name "nbd""* (suggests `nb`, `nsd`).
- `brew info nbdkit` → *"Did you mean nedit?"*; `brew info libnbd` → fuzzy-matches `libbsd`,
  `libnids`, `libnl`, `libb2`, `libmd`, `libdbi`, `libbi`, `libnet`, `libnsl`, `libnfc`,
  `libnfs`, `libccd`, `libabw`, `libwpd`, `libmpd` — none is it; `brew info nbd-server` → absent.
  (All three were already known absent from the spec phase; re-confirmed rather than recalled.)
- PyPI: `nbd` is *"Lightweight ipython/jupyter notebook diffing tool"*; `pyndb` is *"A way of saving
  data to a file with syntactic convenience"*; `libnbd` and `nbdkit` are **not on PyPI** at all.
- Docker is installed but its daemon is not running (`failed to connect to the docker API at
  unix://<docker.sock>`), so a Linux container is not a drop-in either.

**So `qemu-nbd`/`qemu-io` is the cheapest genuinely independent oracle, and installing QEMU is the
recommendation.** One command (`brew install qemu`), 15 deps, 94 GiB free, and it is the strongest
option: `qemu-io` is a real independent *client* and `qemu-nbd` a real independent *server*, so the
differential comparison is independent on both sides.

**One plan claim verified while looking:** the Global Constraints say the oracle's test client is
`github.com/pojntfx/go-nbd`'s `pkg/client` *"only if it is already in go.mod"*. Verified it is
**not**: `grep -n pojntfx go.mod go.sum` → nothing, and the module cache has nothing nbd-related.
So that clause resolves to the in-repo encoder for integration tests, and the oracle's independence
rests entirely on `qemu-io` — which is the correct reading and does not weaken Tier A.

**Cascade recorded:** Task 7's checklist row is *"`oracle_test.go` skips (not fails) without QEMU,
and `run-oracle.sh` treats that skip as a failure"* (plan:2881). Without QEMU installed, Task 7
cannot satisfy its own exit-0 requirement on this machine, and Task 8 modifies the
`test/nbd/README.md` that Task 7 creates — so Task 7 must land before Task 8 regardless.
Also confirmed: `test/nbd/` does not exist yet (`test/crash/device-loss.sh` shows the header
convention Task 7's and Task 8's scripts must follow).

## Task 7 pre-dispatch audit — the oracle's measured traps (commits 3b5cbc13, 80628851, 69ebdd06, 26d2979c, bf95d39e)

Task 7 is the differential oracle against `qemu-nbd`, split into two tiers
(Tier A: in-process Go test serving the real `Handler` and driving it with the
real `qemu-io`; Tier B: the same script against a deployed server when
`DITTOFS_NBD_PORT` is set). The split was ruled in `3ade3fe6` because no live
server can serve this oracle: no command creates a volume
(`grep -rn "volume" cmd/dfsctl/` returns nothing) and the volume is a file
inside a share, so a byte-level oracle also needs an fs-backed local tier.

### QEMU was installed during the audit

`command -v qemu-nbd qemu-img qemu-io` returned nothing at the start of this
audit; `brew install qemu` was run and all three now resolve under
`/opt/homebrew/bin`. `qemu-nbd --version` reports **11.1.2**. Every measurement
below is from that version. The plan previously told the implementer to report
the prerequisite as missing rather than write an unrun script; it now records
that QEMU is installed and Step 4 is runnable.

### FIVE defects found in the oracle's specification, all measured

**1. The transcript diff could never pass (fatal, would have wasted the task).**
The plan required the two `qemu-io` transcripts to "match line for line".
Measured: `qemu-io` prints a timing line after *every* command —
`1 MiB, 1 ops; 00.00 sec (1.533 GiB/sec and 1569.8587 ops/sec)` — and two
**identical** runs against the **same** `qemu-nbd` differ on that line alone:

```
$ diff t1.txt t2.txt
2c2
< 1 MiB, 1 ops; 00.00 sec (1.533 GiB/sec and 1569.8587 ops/sec)
---
> 1 MiB, 1 ops; 00.00 sec (1.389 GiB/sec and 1422.4751 ops/sec)
```

Fix: normalize with `sed '/ops;/d'` before diffing. Measured to make two
identical runs byte-identical. `qemu-img info` is *already* deterministic (no
timing in its output) and needs no normalization — verified by two-run diff.

**2. The exit code is load-bearing and the plan did not mention it.**
`qemu-io` reports a pattern mismatch on **stdout** and exits **1**:

```
$ qemu-io -f raw -c 'read -P 0 0 1M' "$uri"; echo $?     # data is 0x11
Pattern verification failed at offset 0, 1048576 bytes
read 1048576/1048576 bytes at offset 0
1 MiB, 1 ops; 00.00 sec (1.170 GiB/sec and 1197.6048 ops/sec)
1
```

The transcript still prints `read 1048576/1048576 bytes at offset 0`, so a
transcript-only comparison reads a mismatch as success. Fix: emit `exit=$?` into
the transcript after each command and run the sequence with `set +e`.

**This also invalidated the first version of my own probe.** `qemu-io … | tail -1`
showed zeros for both the `--discard=unmap` and no-flag cases, which looked like
the plan's `--discard=unmap` claim was wrong. A negative control (`read -P 0`
against data known to be `0x11`, with no discard) *passed* — proving the probe
could not detect non-zero data at all. Re-run with exit codes, the plan's claim
is confirmed on 11.1.2:

```
WITHOUT --discard=unmap -> read-after-discard exit=1  (discard was DROPPED)
WITH    --discard=unmap -> read-after-discard exit=0  (zeros: discard took effect)
```

Lesson recorded because it is the same failure mode the task exists to prevent:
a probe that cannot fail is not evidence.

**3. `qemu-nbd` exposes an EMPTY export name by default (fatal).**
Measured: with no `-x`, `nbd://127.0.0.1:<port>/x` fails with
`server reported: export 'x' not present`, and even `nbd://…/export/boot` fails
with `server reported: export 'export/boot' not present`. Only the pathless form
works, because it resolves to the empty name. Fix: the oracle's launch must pass
**`-x export/boot`**, making both servers answer to the same name so one URI
string works against both. The asymmetry (the *oracle* needs the flag, the code
under test does not) is what makes it easy to omit.

**4. The flag-parity handshake had four field-layout traps.** Writing it from
memory failed four times against the live server:
- The greeting is **18 bytes**: `NBDMAGIC`(8) + `IHAVEOPT`(8) + `gflags`(u16).
  Reading `>QII` desynchronizes the stream and the next read *hangs* to timeout.
- There is **no version field** in the fixed-newstyle greeting. (An intermediate
  "fix" invented one and broke a working script — see the note below.)
- `gflags` is a capability bitmap: test `gflags & NBD_FLAG_FIXED_NEWSTYLE`, do
  not compare it to a version number.
- `NBD_INFO_EXPORT`'s body is `u16 info_type, u64 size, u16 flags` — unpack
  `">QH"` from `body[2:12]`. `">HQQ"` fails with
  `struct.error: unpack requires a buffer of 18 bytes`.

**5. The plan's own embedded script was wrong, and only running it caught it.**
After embedding the working handshake in the plan, extracting it back out and
running it failed with `TimeoutError` — because the edit that embedded it had
*also* introduced a phantom "read the version after the greeting" step. The
plan's copy and the tested copy had diverged. Re-verified after the fix:

```
$ python3 plan-flags.py 11021 export/boot
flags = 0x0c6d
   0 HAS_FLAGS   2 SEND_FLUSH   3 SEND_FUA   5 SEND_TRIM
   6 SEND_WRITE_ZEROES  10 SEND_CACHE  11 SEND_FAST_ZERO
$ python3 plan-flags.py 11021 wrong/name        # negative control
ValueError: server refused the export (reply type 2147483654)
```

The plan now carries this exact script, marked as verified, with the three traps
annotated inline. DittoFS advertises `0x006d`
(`handshake.go:101`), and `0x006d & 0x0c6d == 0x006d`, so the assertion is
one-directional (ours ⊆ theirs) and never equality — the two extra bits are
`SEND_CACHE | SEND_FAST_ZERO`, which DittoFS deliberately omits.

**A sixth defect, fixed earlier in the same audit:** the numbered "What it does"
list said the `qemu-nbd` scratch file should be "e.g. 64 MiB" while the
construction note required it to be **exactly the export's size**. Fixed to
reference the construction note. The sequence reaches 9 MiB (`write -z 8M 1M`
is the furthest), so a 16 MiB export is the right size.

### The FLUSH metadata gap (Step 0) — a real defect in committed Task 5 code

`handleFlush` called only `h.commit` → `common.CommitBlockStore`, the
**block-store** half of the durability barrier. The metadata half was missing:
`metadata.Service` runs with deferred commits on by default
(`pkg/metadata/service.go:131`), so a WRITE's size and mtime sit in a
pending-writes tracker until something calls `FlushPendingWriteForFile`.
`common.FlushStableWrite` (`internal/adapter/common/write_payload.go:201`) is
exactly `CommitBlockStore` **then** `FlushPendingWriteForFile`, and it is what
NFSv3/v4 WRITE, NFSv3/v4 COMMIT and SMB CLOSE/FLUSH all call. Neither the plan
nor the spec mentioned `FlushStableWrite` anywhere — `grep` returned nothing in
both — so this was a design omission, not an implementer error.

Measured: after `PrepareWrite` + `CommitWrite` + `h.commit` the raw metadata
store's mtime is **unchanged**; calling `FlushPendingWriteForFile(ctx, handle,
false)` afterwards returns `flushed=true` and does change it.

**Ruled Minor, with the severity reasoned rather than asserted:** the size can
never be wrong (NBD's `handleWrite` calls `PrepareWrite` with
`req.Offset+req.Length` and refuses anything past `exp.Size` first, and nothing
in the adapter sets attributes; the flush is max-only), the mtime is not
observable over NBD (the dispatch has five arms and no metadata command), and
the state is bounded (one state per handle, `pkg/metadata/pending_writes.go:83`).
It is fixed anyway because an NBD device is a boot path — Task 8 boots a kernel
from it and a kernel issues `FLUSH` — and the fix is one type widening plus one
body change.

**The fix was ruled, not offered as an option** (`69ebdd06`): the plan had
presented "add the metadata flush **or** document why NBD omits it" while Step 0
said to fix it. The omit-and-document option would have to argue that a client
told "durable" need not be durable, which is the class of bug the residency-truth
gate (spec §7) exists to prevent.

**Two consequences the plan now names, both verified:**
- **The fix breaks the four `connection_test.go` FLUSH tests, with a PANIC, not
  an assertion failure.** `newTestHandler` (`connection_test.go:177`) passes
  `nil` for the metadata service, and `FlushPendingWriteForFile` dereferences it.
  Measured with a temporary probe: `runtime error: invalid memory address or nil
  pointer dereference`. The four tests (`:1119`, `:1139`, `:1161`, `:1177`) move
  onto `newWriteVolumeFixture`, as Tasks 5 and 6 already did for the WRITE and
  punch paths. The EINVAL case keeps asserting `st.flushes == 0`.
- **The type widening is provably safe:** `FlushStableWrite`'s body already
  passes `blockStore` to `CommitBlockStore`, whose parameter is `Committer`
  (`:123`), so the concrete `*engine.Store` must already satisfy it. NFS v3
  (`internal/adapter/nfs/v3/handlers/write.go:299`) and NFS v4
  (`internal/adapter/nfs/v4/handlers/write.go:238`) keep passing a concrete
  `*engine.Store` via `getServicesForHandle` (`internal/adapter/nfs/v3/handlers/utils.go:31`)
  and `resolveBlockStore` (`internal/adapter/nfs/v4/handlers/helpers.go:361`).

### Also corrected in the same pass

- The `storeFake` comment claimed it was "pre-sized to the volume". It is not
  and need not be: `ReadAt` zero-fills both past the written extent and past
  `len(data[id])`. Corrected to say it needs no sizing.
- A Verification Checklist row was added for Step 0, and three more for the
  measured oracle traps (`-x`, normalization, exit codes).

### Dispatch

Task 7 dispatched to agent `worker` (model `cubbit/vllm/mimir`), async run
`del_mv578atf_7v9g`, timeout 150 minutes, base `bf95d39e`, brief 560 lines,
report path `task-7-report.md`. The dispatch carried all five measured traps,
the FLUSH-fix requirements including the nil-meta panic, and an explicit
instruction to report the real `run-oracle.sh` output rather than a claim.

## Task 7 execution — Step 0 landed, the oracle was VACUOUS, and both claims are now falsifiable

**Task 7 ran under two delegate runs.** `del_mv578atf_7v9g` (worker, cubbit/vllm/mimir, 150m) was killed by the watchdog at "no output for 5m" after committing Step 0. `del_mv581vv3_3wwu` (resumed from it, 90m) completed the task. **Lesson reconfirmed: a timed-out async run may have committed real work — check `git log` and `git status` before deciding to re-dispatch, and prefer `resumeFrom`.**

### Step 0 — commit `89ab139c` `fix(nbd): commit the metadata half of the FLUSH barrier`
4 files +94/-17. `internal/adapter/nbd/handlers_control.go:53` is now `return common.FlushStableWrite(h.authContext(ctx, exp), h.meta, bs, exp.Handle, exp.PayloadID, true)`; `internal/adapter/common/write_payload.go` widened to `Committer` (type-only). **Fail-before reproduced exactly as the plan predicted**, verbatim from the delegate's log:
`integration_test.go:475: the raw store's mtime did not move across a FLUSH: 2026-10-12 14:02:38.496738 +0200 CEST -> 2026-10-12 14:02:38.496738 +0200 CEST, so the barrier committed the block store but not the file's pending metadata`
The four `connection_test.go` FLUSH tests moved off `newTestHandler` (nil meta) as the plan required.

### THE FINDING: the differential oracle could not fail
The committed oracle (`1985a7f7`) passed with **`handleDiscard` mutated to `return h.writeReply(conn, req.Cookie, 0, nil)`** — a complete no-op reporting success and freeing nothing. It also passed with **`--discard=unmap` removed** from the `qemu-nbd` launch. Both of its headline claims were unfalsifiable.

Cause, measured: `discard 1M 1M` and `write -z 8M 1M` each ran on a range that had **never been written**, and each was followed by `read -P 0`. A never-written range reads as zeros whether the discard happened or not — `storeFake.ReadAt` zero-fills past the written extent and the qemu-nbd scratch file is sparse — so the comparison agreed in every case including the ones where the code did nothing. `--discard=unmap` was likewise invisible for the same reason: a dropped TRIM leaves zeros, which is what the never-written range already read as.

**Fix `9722cfff` `fix(nbd): make the oracle's discard steps falsifiable`:** a pattern write before each destructive step — `write -P 0x33 1M 1M` before `discard 1M 1M`, `write -P 0x44 8M 1M` before `write -z 8M 1M`. Measured after the fix, BOTH mutations fail with a divergence naming the range and byte count:
- discard no-op → `line 17: qemu-nbd: read 1048576/1048576 bytes at offset 1048576` vs `DittoFS : Pattern verification failed at offset 1048576, 1048576 bytes` (+`exit=1`)
- no `--discard=unmap` → `line 17: qemu-nbd: Pattern verification failed at offset 1048576, 1048576 bytes` vs `DittoFS : read 1048576/1048576 bytes at offset 1048576` (+`exit=0`)

The sequence's highest byte touched is 13631488 (13.0 MiB) against a 16 MiB export — fits. `read -P 0 12M 1M` (never-written) is kept deliberately: it pins that an unwritten range reads as zeros on both servers, the property the two pattern writes must not be confused with.

**This is a general lesson worth carrying:** a differential test over a sparse or zero-filling layer needs every destructive step preceded by a write of a distinguishing pattern. Comparing two servers that both report zeros proves nothing.

### Task 7 deliverables (as committed)
`internal/adapter/nbd/oracle_test.go` (573 lines, **`package nbd`** in-package, real `NewRuntimeExporter` — `fakeExporter.Resolve` is an exact map lookup keyed on `/export/boot` while `qemu-io` sends `export/boot`), `test/nbd/run-oracle.sh` (155 lines), `test/nbd/README.md` (97 lines), and `newIntegrationFixtureAsSized` beside `newIntegrationFixtureAs`. Measured flag words: DittoFS `0x006d`, qemu-nbd 11.1.2 `0x0c6d`, one-directional assertion `ours &^ theirs == 0`. `test/required-tests.json` untouched.

**Environment note:** `gofmt -l .` repo-wide reports `pkg/block/gc/repair_test.go` — **pre-existing at the base commit and not ours** (verified by running gofmt on `git show HEAD:pkg/block/gc/repair_test.go`). Scope gofmt checks to touched files.

## Task 8 audit — the crash rig's `dead_conn_timeout` story is broken on every released nbd-client

Verified by fetching each tag's `nbd-client.c`/`args.c` and grepping, not by trusting dates. **Commits `ff84486c` (plan) and `f91854c9` (spec).**

| | 3.26.1 | 3.27.1 | master |
|---|---|---|---|
| `-T`/`--dead-timeout` | **absent entirely** | present (`args.c:76`, `:198`) | present |
| `NBD_ATTR_DEAD_CONN_TIMEOUT` sent | **never** | behind a gate nothing opens | yes |
| persist over netlink | **no — `-p` ignored** | **no — same bug** | yes |

- **3.26.1** (Debian bookworm `1:3.26.1-6.1`, Ubuntu 24.04 `1:3.26.1-2`): no `-T` at all. Its netlink path does `netlink_configure(...); return 0;` and **never reaches the ioctl reconnect loop**, so `-p`/`--persist` is parsed, stored and silently ignored whenever netlink is available (the default). Its `netlink_configure` sends only `INDEX`, `SIZE_BYTES`, `BLOCK_SIZE_BYTES`, `SERVER_FLAGS` and `TIMEOUT` (`NBD_ATTR_TIMEOUT` = `NBD_SET_TIMEOUT`, a different mechanism).
- **3.27.1** (Debian trixie/sid `1:3.27.1-4`): `nbd-client.c:1548` gates the persist loop on `cur_client->persist_mode`, while `args.c:186-188` sets `client->persist`. **Nothing in the tree ever assigns `persist_mode = true`** — checked `nbd-client.c`, `args.c`, `args.h`, `nbdclt.h`, `nbd.h`, `cliserv.h`; the only two writes are `= false` inside the ioctl path the netlink path returns before. So the loop never starts, the kernel never receives `NBD_CMD_RECONFIGURE`, and a dead connection pauses for `dead_conn_timeout` then returns `BLK_STS_IOERR`. Upstream fixed it on `master`; the fix postdates 3.27.1.

**Harness requirements that follow:** refuse `nbd-client` below 3.27.1 (naming the version found and the distro packages); do **not** pass `-p` on 3.27.1, where `-T` is the only working lever and the restart must fit inside a single window; and assert the window **behaviourally** — SIGKILL the server, confirm guest I/O *blocks* rather than errors — because `dead_conn_timeout` has **no sysfs read-back** (`drivers/block/nbd.c` creates only `pid_attr` and `backend_attr`).

### Everything else the plan claimed about `dead_conn_timeout` was verified TRUE
`args.c:76`/`:199` define and handle `-T`; it reaches the kernel only as `NBD_ATTR_DEAD_CONN_TIMEOUT` (`nbd-client.c:185-186` for master, `:190-191` for 3.27.1); it is absent from the man page; `#else /* HAVE_NETLINK */` defines **empty stubs** for `netlink_configure`, so a non-netlink build genuinely parses and drops it; `check_conn` (`:394`) routes to `netlink_check_conn` only when `use_netlink`, returning 2 on an unresolvable family. Kernel side: `wait_for_reconnect()` (`drivers/block/nbd.c:1101`) opens with `if (!config->dead_conn_timeout) return 0;` — so with the attribute unset the kernel does **not** pause, `nbd_handle_cmd` falls through to `sock_shutdown(nbd)` and returns `BLK_STS_IOERR`. The 2017 commit `560bc4b39952ed77cdb0000992e9415b0ee89edb` is correct, and the kernel has it regardless of the client.

**Also fixed in the plan earlier (`3ade3fe6`):** the `hostfwd=tcp::10809-:10809` launch line was wrong in BOTH documents — `hostfwd` forwards a *host* port *into* the guest, the opposite direction, and would try to bind host port 10809 where DittoFS already listens. Nothing is needed: QEMU's user-net gateway `10.0.2.2` is the host as seen from inside the guest and a guest-initiated outbound connection crosses SLIRP with no rule.

## Task 7 review — the run TIMED OUT at the 45m hard limit, and its activity log yielded four findings

**Dispatch:** reviewer agent, model `cubbit/vllm/mimir`, runId `del_mv597wf3_bds2`, range `bf95d39e..9722cfff` (the two implementation commits; the docs commits in between are mine). Instructed to return the review inline. **It never returned** — exit 143, "timed out: 45m limit", `.out` holds only progress chatter with no verdict.

**This is the second reviewer timeout on this plan.** Both reviewers spent their budget on empirical work rather than reading the diff and writing a verdict. The activity log (`del_mv597wf3_bds2.activity`, 2182 lines, 99 tool calls) does contain real findings, so it was mined rather than discarded. **Process change for Task 8: give the reviewer an explicit tool-call budget (e.g. "at most 25 tool calls") and require the verdict to be the LAST thing written, or accept that empirical reviewers on this plan cannot be trusted to finish.**

**FINDING 1 (real, Important) — `oracleWaitForPort` could never detect a dead `qemu-nbd`.** `internal/adapter/nbd/oracle_test.go:447` read `if cmd.ProcessState != nil && cmd.ProcessState.Exited() { break }`. **`cmd.ProcessState` is nil until `Wait()` returns** — measured with a standalone program: a child that exits immediately (`exit 3`) still reports `ProcessState == nil` on every poll. So a `qemu-nbd` that refused to start cost the full 10s wait and failed with `"qemu-nbd never accepted a connection"`, naming the wrong cause. Fixed in `9ee96beb`: `Wait` runs on a goroutine and closes a channel; the loop `select`s on it and reports `"qemu-nbd exited before accepting a connection"` with stderr. Bite proof: a fake `qemu-nbd` that exits 3 → `FAIL (0.45s) oracle_test.go:161: qemu-nbd exited before accepting a connection on 127.0.0.1:56949; stderr: fake qemu-nbd: refusing to start`.

**FINDING 2 (real, Important, found while fixing 1) — a DATA RACE on the stderr `strings.Builder`.** The timeout branch read `stderr.String()` while the process was still running, racing the pipe copier `Wait` supervises. Measured under `-race` with a fake `qemu-nbd` that loops printing to stderr: `WARNING: DATA RACE` ×2. Fixed in the same commit by killing the process and awaiting the channel before reading. Re-measured after: no race, same failure message.

**FINDING 3 (real, and it corrects the plan) — the oracle does NOT cover FLUSH durability.** Three mutations, each reverted:

| Mutation | `TestOracle_Differential` | `TestIntegration_FlushPersistsPendingMetadata` |
|---|---|---|
| `commit` → `CommitBlockStore` (metadata half dropped) | **PASSES** (`ok 1.745s`) | FAILS |
| `handleFlush` → always `ErrEINVAL` | FAILS | FAILS |
| `handleFlush` → skip the commit, keep the EINVAL check | **PASSES** (`ok 1.496s`) | FAILS |

The oracle catches a FLUSH that answers *wrongly* and misses one that answers `0` without doing anything, because `qemu-io flush` prints the same line either way. The plan had claimed *"the oracle is what makes the durability path observable"* — corrected in `26ae5a64` and `0bc93cfe`, with the table.

**FINDING 4 (NOT a defect — false positive, do not re-inherit).** The reviewer suspected a README/script contradiction about `python3`. Its grep was `grep -n "python3 is used by the transmission-flag check" test/nbd/README.md` → empty, because the README line is backtick-wrapped: `` `python3` is used by the transmission-flag check `` (`test/nbd/README.md:23`). Both files agree; `run-oracle.sh:58` warns and the test skips (`oracle_test.go:357`).

**Verified independently, both clean:** the SKIP-detection regex `grep -qE '^\s*--- SKIP'` (`test/nbd/run-oracle.sh:77`) IS portable on BSD grep 2.6.0-FreeBSD — probed with a literal `s--- SKIP` line and confirmed `\s` is whitespace, not a literal `s`; and there is no import cycle (`go list -deps ./internal/adapter/testruntime | grep -c internal/adapter/nbd` → 0).

**Reviewer's own mutation results, for the record:** `-count=5` on `TestConnection_Flush|TestOracle` → `ok 6.126s` then `ok 7.902s` (no flakiness); TRIM made a no-op → oracle FAILS (reproduced by me independently: `qemu-nbd: / DittoFS : exit=0` divergence, `FAIL 1.593s`); `--discard=unmap` removed from the launch → oracle FAILS. Its "MUT-DROP-1M-ONLY" and "MUT-DROP-4M" runs printed `ok` but were run against a tree where the mutation did not take effect the way it intended (the `qemu-io` sequence writes 1M at offset 0 and 1M at offset 8M, and the drop was keyed on the store's `off`), so those two results are not evidence either way.

## Task 8 pre-dispatch audit — three launch-line defects and the missing volume step (commit 713cb74e)

**The audit method that has worked on every task since Task 4: probe the brief's claims against the real tool or the real code rather than reading them.** QEMU is installed on this machine (`/opt/homebrew/bin/qemu-system-aarch64`, 11.1.2), so the launch line could be run rather than reasoned about.

**DEFECT 1 — `-cpu host` is rejected without an explicit `-accel kvm`.** Measured: `qemu-system-aarch64 -M virt -cpu host -m 2G -display none` → `qemu-system-aarch64: Invalid CPU model: host` followed by `The valid models are: cortex-a7, cortex-a15, cortex-a35, cortex-a55, cortex-a72, cortex-a76, cortex-a710, a64fx, neoverse-n1, neoverse-v1, neoverse-n2, cortex-a53, cortex-a57, max`. Without an accelerator QEMU defaults to TCG, and `host` is meaningless there. `-accel kvm -cpu host` is accepted (verified by running it until the SIGTERM). **Checked the silent-degradation risk too:** `-accel kvm,fallback=allow` on a machine without KVM prints `invalid accelerator kvm` — it does NOT silently fall back, so an explicit `-accel kvm` is safe. This matters because a TCG fallback would still boot and still pass while making every dead-connection-window measurement meaningless. `-accel max` is the portable substitute for `-cpu host`.

**DEFECT 2 — there is no `boot.img`.** The launch line carried `-drive if=virtio,file=boot.img,format=raw`, implying something creates a bootable disk; nothing in the plan does (`grep boot.img` matched only the launch line and its own description). On `-M virt` an arm64 guest has **no BIOS**, and the plan never said how a kernel would be found on a disk. Fixed to `-kernel Image -initrd initramfs.cpio.gz`, verified available: `qemu-system-aarch64 -M virt -kernel /nonexistent` → `could not load kernel '/nonexistent'`, i.e. the option is parsed and acted on. Also added `-display none -serial stdio`, without which the guest transcript — the only evidence the harness produces — never reaches the host, making a successful boot indistinguishable from a hung one.

**DEFECT 3 — nothing provisions the volume, and no command can.** `RuntimeExporter.Resolve` looks up the share root, then `volumes/`, then the volume name. `grep -rn "volume" cmd/dfsctl/` returns **nothing**; `internal/adapter/nbd/volume.go:21` records the intent (*"the operator, not the protocol, decides who owns the volumes"*), so provisioning is by hand through the share's own NFS/SMB mount in production too. The harness must therefore do it explicitly. **The trap inside the fix:** the size must be set *after* creating the file, because `ApplyCreateDefaults` (`pkg/metadata/validation.go:492`) sets `Size = 0` for every non-symlink — `CreateFile` with `FileAttr{Size: N}` does not produce a file of size N. A script that skips provisioning fails at the handshake with `nbd: unknown export` — **the same message an omitted `-N` produces**, which is why Step 5 now requires verifying this prerequisite specifically.

**Also corrected:** the spec's §9.5 topology block described the kernel and initramfs as coming from "firmware" on a local disk — wrong twice over (no firmware on this machine type, no disk in the launch line) — and its fence gained the `&` on the `nbd-client` line, without which a foreground `-p` hangs before `switch_root` (the persist branch returns before `daemon(0,0)`, `nbd-client.c:1571` vs `:1580`).

**Step 6 gained a refusal requirement:** if the host has no KVM the script must refuse rather than fall back to TCG, because a TCG run boots and passes while every timing in it is meaningless. "Not run: no KVM" is the honest outcome; a green TCG run is worse than no run because it looks like evidence.

**Still unresolved and deliberately left so:** the plan does not pin *how* the harness creates `volumes/boot` (NFS mount + `SetFileAttributes`, vs. any other route). The Step 1 text names the requirement and the size trap and says the operator's own route is the model; the implementer will pick one and the review will judge it. Recorded here so the choice is visible as a choice.

---

## After the branch: the review, the pipelining, and the release cleanup

The ledger above ends where the first phase of work ended. Everything after it was
driven by a code review of the finished adapter and by measurement, and it is
recorded here so a reviewer can see what changed after the SDD tasks were closed.

### The review that found two real bugs

A full read of the 14 production files found two defects worth blocking on and
eight minor ones:

**P0 — a WRITE arriving during drain desynchronized the stream.** The draining
branch answered `NBD_ESHUTDOWN` without consuming the payload the client had
already sent, so the next header was parsed out of payload bytes and the
connection died later with `NBD_EBADMAGIC`. That is the opposite of what the
drain exists for. Fixed in `89f2e4a`; the regression test
`TestConnection_DrainingWriteDrainsPayload` was verified to fail without it
(`read reply: read tcp ...: i/o timeout`).

**P1 — the handshake was unbounded.** `ServeHandshake` armed no deadline and the
base adapter arms none on an accepted socket, so a client could connect, send
nothing, and hold a goroutine and a file descriptor forever — before naming an
export, so before any authorization decision exists to fail. Fixed in `69e80b8`.

The minor items became the release cleanup below.

### Pipelining: the ceiling was dispatch, not the store

The measured shape was the argument. Random 4 KiB reads by queue depth reported
the same IOPS at depth 8, 16, 32 and 64, because one goroutine read a header,
handled the command and wrote the reply — the client's queue was only a queue of
waits. A benchmark that isolated the store from the adapter showed **88% of each
request was spent waiting on the store**, behind a dispatcher that could not
overlap anything.

The fix separates the three roles that one loop performed: a reader that is the
only goroutine to advance the stream (a WRITE's payload is inline, so two readers
would consume each other's bytes), a pool of workers that call handlers and never
touch the connection, and a writer that is the only goroutine to write (two
writers would interleave a header and its payload).

The change was made in two commits on purpose. `ad032e42` introduced the pipeline
at **one** worker, which reproduces the serial loop exactly — so the entire
existing suite ran against it unchanged and any behavioural difference would have
been a refactor bug rather than a tuning question. `a56c7016` then raised the
worker count and added the byte budget.

Three things the plan had not anticipated, found by building it:

1. **`Conn.Serve`'s `recover` stopped covering the handler.** It only ever
   protected its own frame, and dispatch left that frame. A panicking handler
   would have killed the process. The worker now recovers per command and answers
   that command with `NBD_EIO`, which is strictly better than the old behaviour —
   the connection survives.
2. **A real data race in `pkg/metadata`.** `RecordWrite` returned a pointer *into*
   the pending-writes map; the caller read it after the lock was released. Harmless
   while dispatch was serialized, a genuine race once it was not. Found by
   `go test -race`, fixed in `pkg/metadata/pending_writes.go`.
3. **A FLUSH barrier had to become explicit.** Serialized dispatch gave the
   ordering guarantee for free. With a pool, a FLUSH can reach a worker while a
   WRITE is still inside the store, so the reader now waits for the count of
   submitted-but-uncommitted writes to reach zero before queueing it.

The in-flight budget is over **bytes, not commands**, and it is taken by READ as
well as WRITE: `handleRead` allocates `req.Length` on the worker, so a
WRITE-only budget would leave N concurrent READs free to allocate N × 32 MiB with
the counter untouched.

Measured result, same host, same fixture, serial build against the pipelined one:

| Depth | Before (serial) | After (4 in flight) |
|---|---|---|
| 1 | 22,075 | 21,141 |
| 8 | 39,682 | 51,282 |
| 16 | 40,160 | 58,139 |
| 32 | 39,215 | 54,945 |
| 64 | 38,167 | 51,020 |

The shape is the finding: before, depth bought nothing above 1; after, depth 16 is
**1.45×** depth 1. Absolute figures on this host vary ±20% between runs and are
not comparable across machines — see [`docs/nbd-benchmarks.md`](../nbd-benchmarks.md).

A real kernel still boots from the pipelined adapter, and the second boot still
recovers the filesystem the first one wrote.

### What was closed from the review

| Item | Resolution |
|---|---|
| `PunchHole` had no request deadline | fixed — `common.WithRequestDeadline` on the punch |
| `FLUSH` accepted undocumented flags | fixed — rejects anything but `FUA` |
| READ honoured no read permission | fixed — `PrepareRead` before the allocation |
| A short read was reported as success | fixed — `NBD_EIO` |
| Export size treated as immutable | fixed — re-resolved, bounded by `size_refresh` |
| `MaxConnections` 0 meant unlimited | fixed — defaults to 64; an uncapped remote bind is refused |
| UID/GID 0 unexpressible | fixed — `uid_set`/`gid_set`, because every uint32 is a legal uid |
| `max_message_size`, `default_export` had no consumers | deleted, rather than wired: the advertised maximum and the enforced maximum must be the same number, and only the enforcement was configurable |
| 10 dead protocol constants + 1 dead codec | deleted |
| No CI coverage | fixed — four NBD gates pinned in `test/required-tests.json`; the oracle now runs in CI with `qemu-nbd` installed |
| No benchmark script | fixed — `test/nbd/bench-fio.sh` |
| Buffer pooling | **still open**, and the byte budget bounds the memory instead |

### Two things this record is careful about

**The oracle's limit.** It catches a FLUSH that answers *wrongly* and misses one
that answers `0` without doing anything, because `qemu-io flush` prints the same
line either way. The table under FINDING 3 above is the evidence; do not read the
oracle as covering the durability path on its own.

**The plan's own errors.** The pipelining plan
(`docs/superpowers/plans/2026-10-20-nbd-pipelining-and-release-cleanup.md`)
claimed at first that handlers would stop touching the socket, which would have
made the reader and `handleWrite` both read the same socket. It also named a
`CmdDiscard` command that does not exist (`handleDiscard` is a helper called by
TRIM and WRITE_ZEROES), and put the FLUSH barrier inside `handleFlush`, where it
would have deadlocked the pool it was part of. All three were corrected before
execution. Where the plan and the code disagreed during execution, the code won,
and the difference is recorded in the commit that resolved it.

---

## The snapshot feature: boot a volume's snapshot, rehydrate it into a new one

The branch's last major piece of work, and the only one that was designed before
it was built. It started from a question about backups — "I want a crash-consistent
snapshot of the whole VM every few minutes" — and the first finding was that
DittoFS already had snapshots. The work was not building snapshots; it was
exposing them to a block-device client.

Design: `docs/superpowers/specs/2026-10-22-nbd-volume-snapshots-design.md`.

### What already existed, and what did not

`Runtime.CreateSnapshot` already produced a share-scoped, crash-consistent copy:
it drains in-flight work, dumps the metadata store, and writes a manifest of the
chunks the dump references. Chunks are content-addressed, so a snapshot costs
little more than what changed. What did not exist was any way for a **client** to
read one, and any way to get one back as a volume.

### The design was reviewed before it was built, and the review mattered

Two flaws in the first draft would each have shipped a broken feature:

**NBD flags are written once, at handshake.** The draft said promotion to
read-write would "clear `NBD_FLAG_READ_ONLY`". `Export.TransmissionFlags()` is
serialized into the greeting and never re-read, so a client that saw the
read-only bit will not send writes no matter what the server does later. Promotion
is a **reconnect**, and the design now says so instead of implying a switch.

**`Export` is not a place to hold mutable state.** `RuntimeExporter.Resolve`
builds a fresh `Export` on every call and `serve` keeps one copy for the
connection's lifetime, so mutating `exp.ReadOnly` changes one connection's view
and nothing re-reads it.

A second review pass found eight more, of which two changed the architecture:

**A private metadata service per snapshot.** Registering the replayed store in
the production service under a synthetic share name forks the lock manager, the
notifier, the quota row and the writeback flag over one namespace
(`pkg/metadata/service.go:1016,1075,1097,1102`). It also cannot be made to work:
file handles are `"<shareName>:<uuid>"` bounded by `MaxFileHandleSize` 64, and a
36-character UUID plus a path is 81 bytes. So a snapshot gets its own
`metadata.Service`, and its handle is minted from the replayed store's own rows.

**The shadow's `RemoteSync` must be built over the replayed store.** Pointing it
at production's store would hydrate production's bytes into a snapshot export —
silently, and only for chunks that were not already local. This is the single
most dangerous wiring mistake in the feature, and the test that catches it is a
byte comparison, not a status check.

### What was built

| Commit | What |
|---|---|
| `725a93e8` | `engine.Store.SetEvictionAllowed` — a read-only store may evict on its own authority. Without it a shadow with a cap fails a large boot with `ErrLocalStoreFull` and there is nothing to evict. |
| `e5494856` | `Runtime.OpenSnapshot` / the open registry, and a delete that refuses a snapshot an export holds (`ErrSnapshotOpen`, 409). |
| `ba94945c` | `OpenSnapshotAsExport`: replay the dump, build the shadow store and its private service. |
| `e2d25328` | The NBD side: `<share>/snapshots/<id>/<volume>` claimed **before** the production split, `Export.Meta`/`store`/`closeFn`, `Handler.metaFor`/`storeFor`. |
| `a4f81be3` | `RehydrateSnapshot` — copy a snapshot into a new, independent, writable volume. |
| `611c8ab9` | The usage report: what each snapshot holds, what a delete frees. |
| `6e9515b5`, `6787aa20` | REST and `dfsctl` for usage and rehydrate. |
| `6d922a21` | The GC-survival test, and two bugs it found. |
| `61768734` | `boot-qemu.sh --export <snapshot shape>`, and the boot that proves it. |

### Bugs the tests found, in the order they were found

**`OpenSnapshotExport.Close` was a data race.** Idempotent is not the same as
concurrency-safe: the connection's deferred release and the test's cleanup both
cleared the same fields. Fixed with a `sync.Once`.

**The rehydrate job published `done` before releasing its hold.** A caller that
polled to `done` and immediately deleted the snapshot was refused with "held open
by 1". `runRehydrate` now *returns* the terminal state; the goroutine releases
the hold, then publishes.

**The usage report filtered snapshots before reading their manifests.** Every
chunk of the reported snapshot then looked unique, so the report claimed a delete
frees everything when it frees nothing. Manifests are always all read now; the
filter applies only to what is reported.

**One volume name cannot identify two volumes.** The first `RehydrateSnapshot`
signature took a single `volume`, which made the snapshot's source and the new
destination the same name. A snapshot is share-scoped and holds every volume in
the share, so the API takes `srcVolume` and `dstVolume`.

**The volume layout was assumed.** Both the shadow resolver and the rehydrate
resolver assumed `<share>/volumes/<volume>`; the test fixtures keep files at the
share root. Both now try `volumes/` and fall back.

### Two traps that made tests pass for the wrong reason

**Three layers refuse a read-only write, so one removal proves nothing.** A write
to a snapshot is refused by `validateWrite`, by `handleWriteZeroes`/`handleDiscard`,
and by `AuthContext.ShareReadOnly` in the metadata layer. The first version of
`TestSnapshotExport_RefusesEveryWrite` passed with the handler's check *deleted* —
because the fixture's volume is owned by uid 1000 and the adapter's identity is
65534, so ordinary ownership denied the write anyway. The test now carries a
positive control: the same identity first writes successfully to the production
export, which proves the identity is authorized before the refusal is attributed
to read-only.

**"GC swept nothing" is also what a broken GC reports.** The GC-survival test
rehydrates, deletes the snapshot, runs GC with a zero grace period, and reads the
volume back — which passes whether the chunks survived or the sweep never ran. It
carries a negative control: deleting the rehydrated volume too must make the same
chunks sweepable (measured 0 swept with the volume present, 2 swept and
2 097 260 bytes freed once it was gone).

### The end-to-end demonstration

Run on 2026-10-25 against a 1 GiB volume whose first boot formatted it and wrote
`hello.txt`:

1. Boot the volume: `VERDICT: PASS`.
2. `Runtime.CreateSnapshot` — the production path — reaches `ready`, 9 chunks.
3. Boot `<share>/snapshots/<id>/boot`: mounts, reads `hello.txt`, `VERDICT: PASS`.
4. Overwrite production's ext4 superblock with 1024 bytes of `0xdeadbeef` at
   offset 1024, over the NBD protocol. Production confirmed unbootable
   (`mount /dev/nbd0 failed`).
5. Boot the same snapshot again: still mounts, still reads `hello.txt`.

Two rig changes were needed and both are properties of snapshots, not workarounds:
the pre-boot `volumes/` check is skipped for a snapshot export (its volume is
deliberately not a file on the share), and the guest mounts `-o noload,ro`
because ext4 will not replay a journal on a read-only device and a snapshot's
journal is dirty for a legitimate reason.

### What this record is careful about

**The shadow store is not itself write-refusing.** A direct `WriteAt` on it
succeeds; the guarantee lives in the handler. Unreachable today because every
write command is refused before the store is touched, but it is defense in depth,
not a property of the store, and it is the layer to re-check if another caller is
ever added.

**Cold reads off a snapshot are slow and were not timed.** Every chunk a boot
touches is hydrated; a single cold 4 KiB read costs 13.75 ms on the S3-backed
host. A full snapshot boot has never been timed, so no figure is claimed.

**Three things are deliberately unbuilt**, each a decision rather than an
omission: cross-share rehydration (needs the dump resolved under another share's
name), byte accounting in the usage report (a manifest line is a hash with no
size; that would need one remote HEAD per distinct chunk), and a `snapshot open`
CLI verb (serving a snapshot is a client connecting to a name, not an operator
action).
