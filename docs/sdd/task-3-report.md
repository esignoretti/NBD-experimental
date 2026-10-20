# Task 3 report — Adapter lifecycle, registration, and the bind gate

**Status:** DONE_WITH_CONCERNS (one brief contradiction found and escalated; resolved with supervisor approval — details below)
**Commit:** `abc387b6b24d1cc0e30ff52d2e0fa5ad39ed58ac` — `feat(nbd): adapter lifecycle, control-plane registration, bind gate`
**Branch:** `nbd-block-protocol` (worktree `<repo>`), base `99f0e834`
**Signed:** yes — SSH signature present (`git cat-file -p HEAD | grep -c gpgsig` → 1). `bash test/no-ai-attribution.sh 99f0e834` → `no AI attribution in 1 commit(s)`.

## Commit

Single commit, conventional-commit form, SSH-signed with the documented invocation:

```
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S \
  -m "feat(nbd): adapter lifecycle, control-plane registration, bind gate"
```

No AI attribution and no `Co-Authored-By` trailer. `bash test/no-ai-attribution.sh 99f0e834` → clean.

## Files

**Created — adapter (`pkg/adapter/nbd/`):**

- `config.go` — `Config`, `TimeoutsConfig`, `DefaultPort`, `DefaultMaxMessageSize`, `DefaultUID`/`DefaultGID`, `applyDefaults`, `validate`.
- `adapter.go` — `Adapter`, `New`, `Serve`, `Stop`, `SetRuntime`, `NewConnection`, plus the placeholder `nbdConn`/`nbdGreeting`.
- `healthcheck.go` — `Healthcheck` override: `Disabled` → delegate.
- `adapter_test.go` (`package nbd_test`) — lifecycle, config validation, bind gate.
- `healthcheck_test.go` (`package nbd_test`) — status mapping.

**Modified:**

- `pkg/controlplane/models/adapter.go` — `DefaultNBDPort = 10809`; `DefaultPort` gains `"nbd"`; `Validate` error text names `nbd`; `Type` field comment updated.
- `cmd/dfs/commands/start.go` — `createAdapterFactory` gains `case "nbd"`; added `createNBDAdapter`.
- `docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md` — Task 3 Step 1 fence: the disconnect test now resolves its runtime through the shared helper, and `conn.Close()` is error-checked. Both edits are the plan-side mirror of changes the supervisor required.

`docs/guide/cli.md` was regenerated with `go run ./cmd/gendocs` (the invocation CLAUDE.md documents; the brief's Step 8 says `go run ./cmd/dfs docs --output docs/guide/cli.md`, which is not the repo's invocation). **It produced no diff** — the NBD adapter adds no CLI command or flag, only an internal factory case — so it is not in the commit. `git status --short docs/guide/cli.md` is empty.

## Step 2 — RED

`go test ./pkg/adapter/nbd/` with only the test file present:

```
github.com/marmos91/dittofs/pkg/adapter/nbd: no non-test Go files in <repo>/pkg/adapter/nbd
FAIL	github.com/marmos91/dittofs/pkg/adapter/nbd [build failed]
```

## Step 7 — GREEN

```
$ go test ./pkg/adapter/nbd/ ./pkg/controlplane/models/ ./cmd/... -v
...
PASS
ok  	github.com/marmos91/dittofs/pkg/adapter/nbd	1.347s
```

31 packages reported `ok`/`no test files`, 0 failures. Per-test results for `pkg/adapter/nbd`:

```
--- PASS: TestNew_AppliesDefaults
--- PASS: TestNew_PanicsOnOutOfRangePort
--- PASS: TestServe_RefusesNonLoopbackBindWithoutOptIn   (#00, 0.0.0.0, ::, 192.168.1.10)
--- PASS: TestServe_RefusesRemoteBindEvenWithNoRuntime
--- PASS: TestServe_AllowsLoopbackBind                   (127.0.0.1, [::1], localhost)
--- PASS: TestServe_AllowsNonLoopbackWithOptIn
--- PASS: TestStop_IsIdempotent
--- PASS: TestServe_ClientDisconnectDoesNotStopTheAdapter
--- PASS: TestControlPlane_DefaultPortKnowsNBD
--- PASS: TestNBDAdapter_Healthcheck_DisabledWhenConfigOff
--- PASS: TestNBDAdapter_Healthcheck_UnknownBeforeStart
```

## TDD bite evidence (deliberate break → assertion failure → exact restore)

Each break was applied to a copy backed up first, the failure observed, then the original restored and verified with `diff` (`RESTORED EXACTLY`).

**1. `TestNew_PanicsOnOutOfRangePort`** — removed the `validate()` call from `New`:

```
--- FAIL: TestNew_PanicsOnOutOfRangePort (0.00s)
    adapter_test.go:33: New with port 70000 did not panic; the adapter constructor contract is to panic on programmer error
```

Fails on the assertion, not a compile error.

**2. Bind gate** — replaced the gate condition with `if false && ...`:

```
    adapter_test.go:58: error "nbd: SetRuntime was not called before Serve" does not name allow_remote_bind, so an operator cannot find the setting
--- FAIL: TestServe_RefusesNonLoopbackBindWithoutOptIn (0.00s)
    --- FAIL: TestServe_RefusesNonLoopbackBindWithoutOptIn/#00 (0.00s)
    --- FAIL: TestServe_RefusesNonLoopbackBindWithoutOptIn/0.0.0.0 (0.00s)
    --- FAIL: TestServe_RefusesNonLoopbackBindWithoutOptIn/:: (0.00s)
    --- FAIL: TestServe_RefusesNonLoopbackBindWithoutOptIn/192.168.1.10 (0.00s)
--- FAIL: TestServe_RefusesRemoteBindEvenWithNoRuntime (0.00s)
```

**3. Gate ordering** — moved the nil-runtime check ahead of the bind gate:

```
    adapter_test.go:73: the runtime check ran before the bind gate: nbd: SetRuntime was not called before Serve
--- FAIL: TestServe_RefusesRemoteBindEvenWithNoRuntime (0.00s)
```

This is the ordering the task calls load-bearing, and the test does pin it.

## What the brief got wrong

**One real contradiction, escalated and resolved.** Step 1's verbatim test fence contains `TestServe_ClientDisconnectDoesNotStopTheAdapter`, which builds the adapter with `nbd.New(cfg)` and calls `a.Serve(ctx)` **without `SetRuntime`**, waiting only on `ListenerReady()`. Step 4 item 2 requires `Serve` to return `"nbd: SetRuntime was not called before Serve"` before it binds, and Task 4's Step 6 preserves that line. The two cannot both hold:

```
--- FAIL: TestServe_ClientDisconnectDoesNotStopTheAdapter (5.00s)
    adapter_test.go:171: listener was never ready
```

The brief's own prose — *"Serve needs a runtime once it gets past the gate, so the listener tests use the real fixture"* — was applied to `serveWithRuntime` but not to this test. The omission is present in every revision of the plan (`cc7b34cb`, `99f0e834`, `d1375c53`, HEAD), so the test predates the check.

I escalated rather than choosing, because it required editing a test the brief presents as verbatim. The supervisor approved keeping the check and fixing the test, and specified the helper rather than inline lines. The test now reads `a, done := serveWithRuntime(t, cfg)`, with the redundant trailing `cancel()`/`<-done` pair dropped (the helper's `t.Cleanup` already cancels and waits). The 300ms `select` asserting `Serve` did **not** return is preserved, and the test is not reordered relative to cleanup. The same edit plus an explanatory comment was applied to the plan fence, as required.

**Two smaller brief/plan inaccuracies:**

- **Step 8's invocation is wrong.** It says `go run ./cmd/dfs docs --output docs/guide/cli.md`; CLAUDE.md's "Frequent commands" says `go run ./cmd/gendocs`, and that is what exists. I used `go run ./cmd/gendocs`.
- **The Step 1 fence fails `errcheck` as written.** `conn.Close()` at the mid-handshake close is unchecked; `GOTOOLCHAIN=go1.26.0 golangci-lint run ./pkg/adapter/nbd/...` reported `pkg/adapter/nbd/adapter_test.go:185:13: Error return value of conn.Close is not checked (errcheck)`. The repo convention is `_ = conn.Close()` (`internal/adapter/nbd/handshake_test.go:76`, `pkg/adapter/nfs/tls_test.go:132`), and the immediately preceding commit `283105a1 test(nbd): check Close errors in the handshake test helper` fixed exactly this class of hit in the previous task's fence. I applied the same fix to both the test file and the plan fence. After the fix: `0 issues`.

Everything else in the brief matched the code. The three "things that will bite you" from the dispatch were all confirmed by reading: `Port()` returns `b.Config.Port` (`pkg/adapter/base.go:598`), the bare-IPv6 listen-address failure comes from `fmt.Sprintf("%s:%d", ...)` at `pkg/adapter/base.go:216`, and `netutil.IsNonLoopbackHost("")` is true via the non-IP fall-through. None of them needed a code change.

## Design notes

- **`Serve` calls `a.BaseAdapter.SetRuntime(rt)` as well as storing `a.runtime`.** The accept loop reaches the base's `Registry` for `RecordConnAccepted`/`RecordConnClosed` (`pkg/adapter/base.go:347`). The dispatch's Interfaces block does not mention this, but the brief's Step 4 describes `SetRuntime` as only "stores it", and a base adapter with a nil `Registry` would panic on the first accepted connection. `Runtime.Metrics()` is nil-receiver-safe, so this is the difference between the metrics path working and panicking.
- **`nbdConn` restates the 18 greeting bytes** rather than exporting `writeGreeting` from `internal/adapter/nbd`. The placeholder is replaced wholesale in Task 4, so widening that package's API for it would be the larger change. The comment says so.
- **`nbdConn.Serve` drains with `io.Copy(io.Discard, c.conn)`.** The base adapter's shutdown sets a 100ms read deadline on active connections, so the drain cannot outlive shutdown. This is what makes `TestServe_ClientDisconnectDoesNotStopTheAdapter` and the loopback tests' cleanup terminate promptly.

## Verification run

| Command | Result |
|---|---|
| `go test ./pkg/adapter/nbd/` (Step 2, RED) | FAIL — no non-test Go files |
| `go test ./pkg/adapter/nbd/ ./pkg/controlplane/models/ ./cmd/... -v` (Step 7) | PASS — 31 packages |
| `go test -race ./pkg/adapter/nbd/ -count=1` | PASS (5.080s) |
| `go build ./...` (Step 9) | clean |
| `make test-unit` (Step 9, `go test -race ./...`, 179 packages) | 139 ok, 5 environmental timeouts — see below |
| `go test -race ./pkg/adapter/nbd/... ./pkg/controlplane/models/... ./cmd/...` | PASS — 31 packages |
| `go test -race -p 1 -timeout 240s ./pkg/adapter/nbd/... ./pkg/controlplane/models/... ./cmd/dfs/...` | PASS — 4 packages |
| `gofmt -l pkg/adapter/nbd/ cmd/dfs/commands/start.go pkg/controlplane/models/adapter.go` | empty |
| `go vet ./pkg/adapter/nbd/...` | clean |
| `GOTOOLCHAIN=go1.26.0 golangci-lint run ./pkg/adapter/nbd/...` | `0 issues.` |

`make test-unit` reported 139 `ok` and 5 `FAIL`, and **all five are `panic: test timed out after 10m0s` in packages this task does not touch**:

```
FAIL	github.com/marmos91/dittofs/internal/adapter/nfs/v3/handlers	609.383s
FAIL	github.com/marmos91/dittofs/internal/adapter/smb/handlers	606.250s
FAIL	github.com/marmos91/dittofs/pkg/block/chunker	600.632s
FAIL	github.com/marmos91/dittofs/pkg/block/engine	601.395s
FAIL	github.com/marmos91/dittofs/pkg/block/middleware/encryption	601.361s
```

These are **resource contention, not regressions**. The `nfs/v3/handlers` stack is stuck in SQLite `AutoMigrate` (`pkg/controlplane/store/gorm.go:433` → `gorm.io/gorm.(*DB).AutoMigrate`) reached from `testing.NewHandlerFixtureWithStore`: many packages each spin up their own SQLite under `-race`, and running them concurrently starves them past the 10-minute deadline. The `engine` timeout is CPU-bound work (`lukechampine.com/blake3/guts.CompressEigentree` under `-race`) getting no scheduling while a dozen packages compete. The controller confirmed this by running the failing packages against the **unmodified base commit** with `-p 1`: `pkg/block/engine` still timed out there, while `pkg/block/middleware/encryption` **passed** serially in 198s after failing at 601s in parallel — so the failure moves with parallelism, not with this diff. None of the five packages imports anything this task changed. `go test -race ./...` was not re-run, per the controller's instruction.

`make lint` was not run: the dispatch documents that the repo-wide target panics on this machine from a Go 1.27/1.26 toolchain mismatch on untouched packages, and `GOTOOLCHAIN=go1.26.0 golangci-lint run ./pkg/adapter/nbd/...` was used instead as instructed.

## Concerns

1. **The plan edit is in the commit.** The supervisor explicitly required the plan fence to carry the same fix so the brief regeneration cannot reintroduce the contradiction. That is why a `docs/` file is in a `feat(nbd)` commit.
2. **`createNBDAdapter` reads `max_connections` as `float64` then `int`.** `AdapterConfig.GetConfig` unmarshals JSON, so a persisted value is always `float64`; the `int` arm covers a directly-set map, matching the pattern at `pkg/controlplane/runtime/init.go:115`. There is no test for `createNBDAdapter` — the brief specifies none, and it is not reachable without a full control-plane fixture. Flagging it as the one piece of this task without direct test coverage.
3. **`DefaultUID`/`DefaultGID` are exported** but the dispatch's Interfaces block lists only the `Config` fields. `applyDefaults` needs them and they document the nobody/nogroup choice; they are additive and harmless, but they are not in the stated contract.
4. **`docs/guide/cli.md` has no diff**, so Step 8's "Expected: `docs/guide/cli.md` updated; `git diff --stat` shows it" does not hold. The reason is that NBD adds no user-visible CLI surface — only an internal factory case — so `go run ./cmd/gendocs` rewrote the file identically. An unchanged file is the correct outcome here; it was not staged to look busy.
5. **`pkg/adapter/nbd/` was briefly stashed mid-run by the controller** (to run the failing packages against the clean base) and restored immediately. The final tree and commit were verified after restoration; nothing was lost. Worth recording because a reader seeing `git stash` activity in a shared worktree should know it was deliberate and reversed.

## Closing

Step 10 complete: commit `abc387b6`, working tree clean, nothing staged. `go build ./...` clean from the committed state.

---

# Task 3 review fixes

**Status:** DONE
**Commit:** `d8e5e8b852cb7ae2b5abd5d233079a22f7aa2078` — `fix(nbd): bind listener tests to a free port and stop the double done read` (SSH-signed, no AI attribution, working tree clean).
**Scope:** exactly the three findings below. Nothing else was restructured; the bind gate, its ordering, the runtime check, the control-plane wiring, and the deliberate absence of a `handler` field are unchanged.

## Fix 1 — `cfg.Port = 0` did not mean ephemeral; the listener tests bound the fixed 10809

**Confirmed before fixing.** `Config{Port: 0} -> a.Port() = 10809 (DefaultPort = 10809)`, because `applyDefaults` rewrites `<= 0` at `config.go:102` before `adapter.go:43` hands the port to `NewBaseAdapter`.

**Consequence reproduced** by holding 10809 with a listener:

```
--- FAIL: TestServe_AllowsLoopbackBind/127.0.0.1
    adapter_test.go:116: Serve returned before the listener was ready: failed to create nbd listener on port 10809: listen tcp 127.0.0.1:10809: bind: address already in use
--- FAIL: TestServe_ClientDisconnectDoesNotStopTheAdapter
    adapter_test.go:171: listener was never ready
```

**Fixed the tests, not `applyDefaults`.** Added the `freePort` helper (same shape as `pkg/controlplane/api/tls_test.go:192`) and used `cfg.Port = freePort(t)` in all three listener tests: `TestServe_AllowsLoopbackBind`, `TestServe_AllowsNonLoopbackWithOptIn`, `TestServe_ClientDisconnectDoesNotStopTheAdapter`. `applyDefaults` was left alone: an explicit 0 binds the default port in production, which is what `Config.Port`'s doc at `config.go:64` says.

The two comments asserting an ephemeral mechanism that cannot occur (`:111`, `:119-124`) and the dial comment (`:174-175`) were rewritten to describe what the code now does.

**Proof the fix bites — both runs, with 10809 held by a separate listener process the whole time:**

```
# freePort in place
--- PASS: TestServe_AllowsLoopbackBind (0.07s)
--- PASS: TestServe_AllowsNonLoopbackWithOptIn (0.02s)
--- PASS: TestServe_ClientDisconnectDoesNotStopTheAdapter (0.32s)
ok  	github.com/marmos91/dittofs/pkg/adapter/nbd	1.632s

# reverted to cfg.Port = 0, 10809 still held
    adapter_test.go:142: Serve returned before the listener was ready: ... address already in use
--- FAIL: TestServe_AllowsLoopbackBind (0.07s)
    adapter_test.go:197: listener was never ready
--- FAIL: TestServe_ClientDisconnectDoesNotStopTheAdapter (5.02s)
FAIL
```

Then restored and re-verified: `grep -c "cfg.Port = freePort(t)"` → 3, `grep -c "cfg.Port = 0"` → 0.

## Fix 2 — `validate()`'s comment described an unreachable state, and one arm was dead

The comment claimed "Port 0 is valid: it asks the OS for an ephemeral port, which is what the lifecycle tests use" — contradicting `Config.Port`'s own doc, and unreachable because `applyDefaults` rewrites any `<= 0` first. That made `c.Port < 0` dead too: a negative becomes 10809 before `validate` runs.

Dropped the dead arm; `validate()` is now `if c.Port > 65535`. The comment states the truth: `validate` has exactly one caller (`adapter.go:45`, immediately after `applyDefaults`), so a port here is always in range or too large, and the negative case is unreachable rather than merely unlikely. `TestNew_PanicsOnOutOfRangePort` covers the reachable arm.

## Fix 3 — `serveWithRuntime`'s `done` channel was read twice

Confirmed on the failure path before fixing. The helper's cleanup read `done`, and the test body also read it at `adapter_test.go:116` and `:180`. When the body consumed the value first, cleanup blocked its full 5 seconds and then reported a second, bogus error for the same cause:

```
adapter_test.go:116: Serve returned before the listener was ready: ... address already in use
adapter_test.go:92:  Serve did not return after context cancellation
```

`Serve` had returned; the body already took the value. Gave cleanup its own `served` channel (closed by the serve goroutine after it sends on `done`), so cleanup no longer steals the value the body needs. The body's `done` reads are unchanged.

**Verified both halves:** the bogus `adapter_test.go:92` line is gone from the reverted-port run above, and that run's `TestServe_AllowsLoopbackBind` now fails in **0.07s** instead of stalling 10.07s — one cause, one error, no false 5-second wait.

## Plan updated

The same three test bodies, the same `freePort` helper, and the same `serveWithRuntime` change are in `docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md` (Task 3, Step 1's fence). Step 4's `validate()` prose changed from `Port < 0 || Port > 65535` to `Port > 65535`. The "ephemeral" claim is gone from the plan: `grep -n "ephemeral"` returns only the `freePort` doc comment explaining why port 0 cannot be used. Verified the fence is byte-identical to the implementation apart from the `io` import the fence intentionally omits.

## Verification

| Command | Result |
|---|---|
| `go test -race ./pkg/adapter/nbd/... -count=1` | PASS (9.436s) |
| `go build ./...` | clean |
| `gofmt -l pkg/adapter/nbd/` | empty |
| `go vet ./pkg/adapter/nbd/...` | clean |
| `GOTOOLCHAIN=go1.26.0 golangci-lint run ./pkg/adapter/nbd/...` | `0 issues.` |
| `go test ./pkg/adapter/nbd/ ./pkg/controlplane/models/ ./cmd/...` | `pkg/adapter/nbd`, `pkg/controlplane/models` and the rest PASS; **one pre-existing failure** — see below |

## One pre-existing failure found, not fixed (outside the three)

`go test ./cmd/...` now fails in `cmd/gendocs`:

```
--- FAIL: TestDocsCommandReferences (0.07s)
    helpref_test.go:407: 1 stale command reference(s) — the command moved or the flag was renamed:
        docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md:1352: `dfs docs` — dfs has no subcommand "docs"
```

**It is not mine and it is not in the three findings.** `git show HEAD:...` shows the phrase originates in the controller's own `a0cc7e20 docs(nbd): correct task 3's docs step and record the environmental test timeouts`, whose text reads *"There is no `dfs docs` subcommand"* — a correct sentence that the reference checker matches on the backticked command alone, without reading the negation. I confirmed it fails on unmodified HEAD too, by stashing my work and running the test against the committed tree:

```
# on HEAD with my changes stashed
--- FAIL: TestDocsCommandReferences (0.13s)
        docs/superpowers/plans/2026-10-08-dittofs-nbd-block-protocol.md:1325: `dfs docs` ...
FAIL	github.com/marmos91/dittofs/cmd/gendocs	1.361s
```

I left it alone: it is a `docs/`-only, one-line fix in a file the review did not flag, and "fix these three and nothing else" was explicit. Flagging it because it means `go test ./cmd/...` is red on this branch for a reason unrelated to Task 3.

## Deferred items left untouched, as instructed

`createNBDAdapter`'s four-key config read, the `UID == 0`-is-unexpressible ambiguity, `UpdateAdapter`'s listener preservation, the `#00` subtest rendering, the repo-wide `make lint` panic, and the five `make test-unit` timeouts — none were touched.
