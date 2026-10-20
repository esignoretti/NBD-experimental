# Task 5 Review — WRITE / FLUSH / FUA (6fa295e3, base 7b284dfc)

Reviewer: agent `reviewer`, model `cubbit/vllm/mimir`, run `779e8e47-1535-4695-870f-28a05fba84c4`.
Persisted by the controller: the `reviewer` agent has no write tool, so it returned
the review inline and the controller wrote this file. The text below is the
reviewer's, verbatim.

## Spec Compliance

1. **Check order + subtraction overflow — ✅ met.** `internal/adapter/nbd/handlers_write.go:33` flags (`req.Flags&^FlagFUA != 0` → EINVAL) → `:38` `exp.ReadOnly` → EPERM → `:46` `req.Length == 0` → EINVAL, no drain → `:52` `req.Offset > math.MaxUint64-uint64(req.Length)` → EIO (subtraction, not addition) → `:60` `req.Offset+uint64(req.Length) > exp.Size` → ENOSPC → `:68` payload read → `:86` `PrepareWrite` → `:93` `bs.WriteAt` → `:100` `CommitWrite` → `:107` FUA commit → `:113` reply. Matches the required order exactly.
2. **Desync guard — ✅ met.** `rejectWrite` (`handlers_write.go:133` reply, then `:144` `io.CopyN(io.Discard, conn, int64(req.Length))`) — bounded buffer, never `make([]byte, req.Length)`. Reply precedes drain (deliberate; opposite directions). Zero-length path (`:46-48`) uses `writeReply` only, no drain. All non-zero-Length rejections route through `rejectWrite`.
3. **FUA ordering — ✅ met.** `handlers_write.go:107-111`: `h.commit(...)` runs before `:113` `writeReply(..., 0, nil)`.
4. **`commit` delegates — ✅ met.** `handlers_control.go:49` `return common.CommitBlockStore(ctx, bs, exp.PayloadID)`; no second copy of the durability rule anywhere in non-test `internal/adapter/nbd/` (grep for `RequireDurableCommit|LocalDurable|RemoteDurable|ErrNotDurableYet` returns only test/comment hits). `common/write_payload.go:123` is the shared NFS/SMB seam.
5. **`mapError` — ✅ met.** `connection.go:301` normalizes via `common.NormalizeBlockStoreError` before classifying; `:333` `case merrs.ErrPermissionDenied, merrs.ErrAccessDenied:` → EPERM. Only `ErrEIO`/`ErrEPERM`/`ErrENOSPC`/`ErrESHUTDOWN`/0 are returned (`connection.go:314,319,327,340,347,353`); `protocol.go:80-84` defines no EOVERFLOW/ENOTSUP. `ErrAccessDenied` is real: `pkg/metadata/auth_permissions.go:360,510,539` return it for mode-bit denials, so the arm is load-bearing.
6. **FUA durability test mechanism — ✅ met.** `connection_test.go` FUA test installs `st.onFlush` (set pre-goroutine, `storeFake.Flush` invokes it), parks inside `Flush`, then reads the reply with a 250 ms deadline while parked and fails if any header arrives (`sawReply`). Moving `commit` after the reply makes the reply readable while parked → test fails. The assertion is on ordering, not on `st.flushes` alone.
7. **Stream-sync test — ✅ met.** `TestConnection_RejectedWriteKeepsTheStreamSynchronized` sends the full 4096-byte payload for an out-of-range WRITE (ENOSPC), then a valid READ at offset 0 and asserts `Error == 0` and byte equality with the pre-seeded `0x5a` payload.
8. **Real metadata for happy-path WRITEs — ✅ met.** `writeVolumeFixture.newHandler` (`integration_test.go:69-73`) passes `f.rt.GetMetadataService()`; fixture creates share root + volume as `integrationIdentity{65534,65534}` (`integration_test.go:35, 96-131`) matching the handler's identity. The deliberate mismatch exists only in `TestIntegration_WriteDeniedByPermissionIsEPERM` (created as 1000/1000, served as 65534).

## Strengths

- `TestConnection_RejectedHugeWriteDoesNotAllocate` asserts `TotalAlloc < declared` **and** that serve returns non-nil after the drain deadline — it proves bounded-buffer draining rather than just claiming it.
- `TestConnection_WriteZeroLengthIsEINVAL` proves the no-drain decision functionally: the following READ must still succeed on the same connection.
- Integration tests assert preconditions (`RequireDurableCommit`, `LocalDurable`, `RemoteDurable`) instead of assuming them, and the durability-override lever matches `internal/adapter/common/commit_durability_test.go`.

## Issues

- **Minor** `connection_test.go` (`TestConnection_RejectedHugeWriteDoesNotAllocate`): the `runtime.MemStats.TotalAlloc` delta is process-global; it would be flaky if any test in the package ever calls `t.Parallel()`. No current test does, so this is a latent robustness note only.
- **Minor** `handlers_write.go:69`: `io.LimitReader(conn, int64(req.Length))` is redundant with the exact-size `data` slice passed to `io.ReadFull`. Harmless; no fix needed.
- No Critical or Important issues.

## Assessment

**Task quality: Approved.** All eight checks are met with file:line evidence: the required check order and subtraction-based overflow guard are exact, the desync guard drains only non-zero-length rejections with a bounded `io.CopyN`, the reply-before-drain choice is documented and defensible, FUA commits before the reply, and `commit` is a one-line delegation to `common.CommitBlockStore` with no duplicated durability rule. `mapError` normalizes first and adds the `ErrAccessDenied → EPERM` arm that the metadata permission funnel actually returns. The tests pin mechanisms, not outcomes — the FUA test parks the server inside `Flush` and would fail if the commit moved after the reply, and the stream-sync and zero-length tests both prove stream position with a subsequent READ. The two minor notes are report-only.

Residual risk: I did not run the suite (per instruction); the 58 PASS / 0 FAIL claim rests on the implementer's report and the controller's two bite proofs, which the test mechanisms above corroborate.
