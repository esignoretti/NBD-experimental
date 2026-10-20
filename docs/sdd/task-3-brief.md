## Task 3: Adapter lifecycle, registration, and the bind gate

**Files:**
- Create: `pkg/adapter/nbd/config.go`, `pkg/adapter/nbd/adapter.go`, `pkg/adapter/nbd/healthcheck.go`
- Modify: `pkg/controlplane/models/adapter.go` (`DefaultPort` at `:34`, `Validate` at `:50`), `cmd/dfs/commands/start.go` (`createAdapterFactory` at `:900`)
- Test: `pkg/adapter/nbd/adapter_test.go`, `pkg/adapter/nbd/healthcheck_test.go`

**Interfaces:**
- Consumes: Task 2's `ServeHandshake`, `RuntimeExporter`, `Exporter`, `Identity`. **Nothing from Task 4.** An earlier draft had this task's `Serve` build the `nbd.Handler`, which made this task's commit uncompilable until Task 4 landed; the wiring now lives in Task 4's Step 6, where every symbol it names already exists. This task binds and delegates.
- Produces:
  - `type TimeoutsConfig struct { Read, Write, Idle, Shutdown time.Duration }` — the operator-facing four-timer config. The wire layer's three-timer `nbd.Timeouts` and the `timeouts()` narrowing accessor belong to Task 4, which is where they are first needed; this task only stores the config.
  - `type Config struct { Enabled bool; Port int; BindAddress string; MaxConnections int; MaxMessageSize int; Timeouts TimeoutsConfig; AllowRemoteBind bool; DefaultExport string; UID uint32; GID uint32 }`
  - `const DefaultPort = 10809`
  - `func (c *Config) applyDefaults()`, `func (c *Config) validate() error`
  - `type Adapter struct { *adapter.BaseAdapter; config Config; runtime *runtime.Runtime }` — no `handler` field in this task; Task 4's Step 6 adds it together with the wiring. Declaring it here is what made an earlier draft uncompilable on its own.
  - `func New(config Config) *Adapter`
  - `func (a *Adapter) Serve(ctx context.Context) error`
  - `func (a *Adapter) Stop(ctx context.Context) error`
  - `func (a *Adapter) SetRuntime(rt any)`
  - `func (a *Adapter) NewConnection(conn net.Conn) adapter.ConnectionHandler`
  - `func (a *Adapter) Healthcheck(ctx context.Context) health.Report`

- [ ] **Step 1: Write the failing config and lifecycle tests**

Create `pkg/adapter/nbd/adapter_test.go`. **It is `package nbd_test`** (external), matching `pkg/adapter/nfs/healthcheck_test.go` and `pkg/adapter/smb/healthcheck_test.go`, both of which are `_test` packages. That choice is load-bearing here: the tests need `internal/adapter/testruntime`, whose package doc records that *"Every consumer is an in-package test, so importing the package under test would close an import cycle"* — an external test package breaks that constraint. Verified for this package: `go list -deps ./internal/adapter/testruntime` contains no `dittofs/pkg/adapter` entry, so `pkg/adapter/nbd` (imported by `package nbd_test`) creates no cycle. Do not "simplify" this to `package nbd` to reach an unexported symbol; add an exported accessor instead.

```go
package nbd_test

import (
	"context"
	"net"
	"strings"
	"testing"
	"time"

	"github.com/marmos91/dittofs/internal/adapter/testruntime"
	"github.com/marmos91/dittofs/pkg/adapter/nbd"
	"github.com/marmos91/dittofs/pkg/controlplane/models"
)

func minimalConfig() nbd.Config {
	return nbd.Config{Enabled: true, Port: 11089, BindAddress: "127.0.0.1"}
}

func TestNew_AppliesDefaults(t *testing.T) {
	a := nbd.New(nbd.Config{})
	if a.Port() != nbd.DefaultPort {
		t.Fatalf("Port() = %d, want %d", a.Port(), nbd.DefaultPort)
	}
	if a.Protocol() != "nbd" {
		t.Fatalf("Protocol() = %q, want %q", a.Protocol(), "nbd")
	}
}

func TestNew_PanicsOnOutOfRangePort(t *testing.T) {
	defer func() {
		if recover() == nil {
			t.Fatal("New with port 70000 did not panic; the adapter constructor contract is to panic on programmer error")
		}
	}()
	nbd.New(nbd.Config{Enabled: true, Port: 70000})
}

// The bind gate. An NBD export is unauthenticated by construction: there is no
// AUTH_SYS, no Kerberos, and a single READ hands over every byte with no path
// scoping. A non-loopback bind must therefore be an explicit opt-in.
//
// The empty string is in the list deliberately: IsNonLoopbackHost("") is true
// (net.ParseIP("") is nil and the final branch assumes a hostname resolves
// off-host), which is the behaviour the gate wants — an empty BindAddress binds
// every interface. Do not add a special case for it.
func TestServe_RefusesNonLoopbackBindWithoutOptIn(t *testing.T) {
	for _, bind := range []string{"", "0.0.0.0", "::", "192.168.1.10"} {
		t.Run(bind, func(t *testing.T) {
			cfg := minimalConfig()
			cfg.BindAddress = bind
			a := nbd.New(cfg)
			err := a.Serve(context.Background())
			if err == nil {
				t.Fatal("Serve on a non-loopback address without AllowRemoteBind returned nil")
			}
			if !strings.Contains(err.Error(), "allow_remote_bind") {
				t.Fatalf("error %q does not name allow_remote_bind, so an operator cannot find the setting", err)
			}
		})
	}
}

// The gate runs before the listener is created and before the runtime is
// consulted, so this test needs neither a free port nor a fixture.
func TestServe_RefusesRemoteBindEvenWithNoRuntime(t *testing.T) {
	cfg := minimalConfig()
	cfg.BindAddress = "0.0.0.0"
	a := nbd.New(cfg)
	if err := a.Serve(context.Background()); err == nil {
		t.Fatal("the bind gate must refuse before Serve touches the runtime")
	} else if strings.Contains(err.Error(), "SetRuntime") {
		t.Fatalf("the runtime check ran before the bind gate: %v", err)
	}
}

// Serve needs a runtime once it gets past the gate, so the listener tests use
// the real fixture. This also exercises SetRuntime, which nothing else does.
func serveWithRuntime(t *testing.T, cfg nbd.Config) (*nbd.Adapter, chan error) {
	t.Helper()
	rt, _ := testruntime.NewWithShareStore(t)
	a := nbd.New(cfg)
	a.SetRuntime(rt)
	ctx, cancel := context.WithCancel(context.Background())
	done := make(chan error, 1)
	go func() { done <- a.Serve(ctx) }()
	t.Cleanup(func() {
		cancel()
		select {
		case <-done:
		case <-time.After(5 * time.Second):
			t.Error("Serve did not return after context cancellation")
		}
	})
	return a, done
}

func TestServe_AllowsLoopbackBind(t *testing.T) {
	// "::1" is deliberately absent and "[::1]" stands in for it. BaseAdapter
	// builds its listen address as fmt.Sprintf("%s:%d", BindAddress, Port)
	// (pkg/adapter/base.go:216), so a BARE IPv6 literal yields "::1:0" and
	// net.Listen rejects it with "too many colons in address"; the bracketed
	// form is what binds. That is a property of the shared base adapter, not
	// something this task changes, so the test does not assert a capability the
	// code lacks. Both spellings pass netutil.IsNonLoopbackHost as loopback, so
	// the gate admits either — only the listen address differs.
	for _, bind := range []string{"127.0.0.1", "[::1]", "localhost"} {
		t.Run(bind, func(t *testing.T) {
			cfg := minimalConfig()
			cfg.BindAddress = bind
			cfg.Port = 0 // ephemeral: the test must not need a fixed free port
			a, done := serveWithRuntime(t, cfg)
			select {
			case <-a.ListenerReady():
			case err := <-done:
				t.Fatalf("Serve returned before the listener was ready: %v", err)
			case <-time.After(5 * time.Second):
				t.Fatal("listener was never ready")
			}
			// GetListenerAddr, not Port. Port() returns the CONFIGURED port
			// (pkg/adapter/base.go:598 is "return b.Config.Port") and nothing ever
			// writes the bound port back into Config, so Port() is legitimately 0
			// whenever the test asked for an ephemeral one. GetListenerAddr reports
			// the socket's real address, and it blocks on listenerReady too.
			if a.GetListenerAddr() == "" {
				t.Fatal("GetListenerAddr is empty after ListenerReady; the adapter did not bind a listener")
			}
		})
	}
}

func TestServe_AllowsNonLoopbackWithOptIn(t *testing.T) {
	cfg := minimalConfig()
	cfg.BindAddress = "0.0.0.0"
	cfg.AllowRemoteBind = true
	cfg.Port = 0
	a, done := serveWithRuntime(t, cfg)
	select {
	case <-a.ListenerReady():
	case err := <-done:
		t.Fatalf("Serve with AllowRemoteBind returned early: %v", err)
	case <-time.After(5 * time.Second):
		t.Fatal("listener was never ready")
	}
}

func TestStop_IsIdempotent(t *testing.T) {
	a := nbd.New(minimalConfig())
	ctx := context.Background()
	if err := a.Stop(ctx); err != nil {
		t.Fatalf("first Stop: %v", err)
	}
	if err := a.Stop(ctx); err != nil {
		t.Fatalf("second Stop: %v", err)
	}
}

// The adapter must not treat a per-connection failure as fatal: Serve returning
// before cancellation makes DittoServer stop every other adapter.
func TestServe_ClientDisconnectDoesNotStopTheAdapter(t *testing.T) {
	cfg := minimalConfig()
	cfg.Port = 0
	a := nbd.New(cfg)
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	done := make(chan error, 1)
	go func() { done <- a.Serve(ctx) }()
	select {
	case <-a.ListenerReady():
	case <-time.After(5 * time.Second):
		t.Fatal("listener was never ready")
	}

	// Dial the socket's real address rather than the configured port: this test
	// asks for an ephemeral port, and Port() reports the configured value.
	addr := a.GetListenerAddr()
	for range 3 {
		conn, err := net.Dial("tcp", addr)
		if err != nil {
			t.Fatalf("dial: %v", err)
		}
		// Read the greeting, then hang up mid-handshake.
		buf := make([]byte, 18)
		_, _ = io.ReadFull(conn, buf)
		conn.Close()
	}

	select {
	case err := <-done:
		t.Fatalf("Serve returned after client disconnects: %v", err)
	case <-time.After(300 * time.Millisecond):
	}
	cancel()
	<-done
}

func TestControlPlane_DefaultPortKnowsNBD(t *testing.T) {
	if got := models.DefaultPort("nbd"); got != nbd.DefaultPort {
		t.Fatalf("models.DefaultPort(\"nbd\") = %d, want %d", got, nbd.DefaultPort)
	}
	if err := (&models.AdapterConfig{Type: "nbd", Port: 10809}).Validate(); err != nil {
		t.Fatalf("Validate rejected an nbd adapter: %v", err)
	}
}
```

Add the missing import (`io`) at the top.

- [ ] **Step 2: Run it and confirm it fails**

Run: `go test ./pkg/adapter/nbd/ -v`
Expected: FAIL — package does not exist.

- [ ] **Step 3: Implement `config.go`**

Mirror `pkg/adapter/smb/config.go:63` and its `applyDefaults` at `:311`. Defaults: `Port` 10809; `Timeouts.Read` 5m, `Write` 30s, `Idle` 5m, `Shutdown` 30s; `MaxMessageSize` 32 MiB (`32 << 20` — the maximum block size we advertise); `UID`/`GID` `65534` (nobody/nogroup, the unprivileged default the SMB synthetic-identity path uses, `internal/adapter/smb/handlers/auth_helper.go:74-95`).

`validate()` returns an error when `Port < 0 || Port > 65535`. `New` panics on a `validate()` error, matching `pkg/adapter/smb/adapter.go:176`.

- [ ] **Step 4: Implement `adapter.go`**

`New` calls `applyDefaults()`, `validate()` (panic on error), then `adapter.NewBaseAdapter(adapter.BaseConfig{BindAddress, Port, MaxConnections, ShutdownTimeout: Timeouts.Shutdown}, "nbd")`.

`SetRuntime(rt any)` type-asserts to `*runtime.Runtime` and stores it. It must **not** build the exporter or the handler: `nbd.NewHandler` and `nbd.Identity` are declared by Task 4, so referencing them here would leave this task's own commit uncompilable. Task 4's Step 6 adds the wiring. This task's job is the lifecycle, the bind gate, and the control-plane registration.

`Serve(ctx)`:
1. **The bind gate, before anything binds.** `if netutil.IsNonLoopbackHost(a.config.BindAddress) && !a.config.AllowRemoteBind { return fmt.Errorf(...) }`. The error text must contain `allow_remote_bind`. `netutil.IsNonLoopbackHost` (`pkg/netutil/netutil.go:15`) returns false for `127.0.0.1`, `::1`, `[::1]`, `localhost`, and **true for an empty string** (it falls through the switch, `net.ParseIP("")` is nil, and the non-IP branch assumes off-host). That is the behaviour the gate wants — an empty `BindAddress` binds all interfaces, so it must be refused without the opt-in — and it needs no special case. Say so in a comment, because the next reader will not check the fall-through. The gate runs **before** the runtime is consulted, which is what `TestServe_RefusesRemoteBindEvenWithNoRuntime` pins; Task 4's Step 6 preserves that order when it fills in the body.
2. `if a.runtime == nil { return errors.New("nbd: SetRuntime was not called before Serve") }`.
3. `return a.ServeWithFactory(ctx, a, nil, nil)`.

`NewConnection(conn net.Conn) adapter.ConnectionHandler` returns a minimal per-connection type implementing `Serve(ctx context.Context)` that reads the greeting and closes, so the lifecycle tests pass. It exists because `adapter.ConnectionHandler.Serve` takes only a context (`pkg/adapter/base.go:38`), so the socket has to be bound at construction. **Task 4's Step 6 replaces this stub** with `return nbd.NewConn(a.handler, conn)`, the same shape as `pkg/adapter/smb/adapter.go:717`. (Task 4 lists `pkg/adapter/nbd/adapter.go` under Modify for exactly this reason.)

`Stop` delegates to `a.BaseAdapter.Stop(ctx)`.

- [ ] **Step 5: Implement `healthcheck.go`**

```go
func (a *Adapter) Healthcheck(ctx context.Context) health.Report
```

`!a.config.Enabled` → `StatusDisabled` with message `"NBD adapter is disabled in config"`. Otherwise delegate to `a.BaseAdapter.Healthcheck(ctx)`. `StatusDegraded` is added in Task 4, when there is a resolvable-export failure to report — until then there is nothing that can degrade.

Note `BaseAdapter.Healthcheck` already reports `StatusUnhealthy` once shutdown is initiated but `Serve` has not returned (`pkg/adapter/healthcheck.go:39`), so the drain path needs no help here.

- [ ] **Step 6: Wire the control plane**

In `pkg/controlplane/models/adapter.go`:
- Add `DefaultNBDPort = 10809` beside `DefaultNFSPort`/`DefaultSMBPort` at `:28`.
- Add `case "nbd": return DefaultNBDPort` to `DefaultPort` at `:34`.
- Change the `Validate` error text at `:55` to `"unsupported adapter type %q: must be one of nfs, smb, nbd"`.
- Update the `Type` field comment at `:9` from `// nfs, smb` to `// nfs, smb, nbd`.

In `cmd/dfs/commands/start.go`, add to `createAdapterFactory`'s switch at `:902`:

```go
case "nbd":
    return createNBDAdapter(cfg)
```

and add `createNBDAdapter`, modelled on `createSMBAdapter` at `:950`: resolve `port := cfg.Port; if port == 0 { port = models.DefaultNBDPort }`, build `nbdCfg := nbd.Config{Enabled: true, Port: port}`, call `cfg.GetConfig()` (error `"failed to parse adapter config: %w"`), then read `bind_address`, `allow_remote_bind`, `default_export`, `max_connections` out of the `map[string]any`.

- [ ] **Step 7: Run the tests and confirm they pass**

Run: `go test ./pkg/adapter/nbd/ ./pkg/controlplane/models/ ./cmd/... -v`
Expected: PASS.

- [ ] **Step 8: Regenerate the CLI docs**

Run: `go run ./cmd/dfs docs --output docs/guide/cli.md` (check the exact invocation in CLAUDE.md's "Frequent commands" before running).
Expected: `docs/guide/cli.md` updated; `git diff --stat` shows it.

- [ ] **Step 9: Run the full unit suite and build**

Run: `go build ./... && make test-unit`
Expected: both clean. `make test-unit` is `go test -race ./...` — the lifecycle tests must be race-clean.

- [ ] **Step 10: Commit**

```bash
git add pkg/adapter/nbd/ pkg/controlplane/models/adapter.go cmd/dfs/commands/start.go docs/guide/cli.md
git -c gpg.format=ssh -c user.signingkey=$HOME/.ssh/id_rsa.pub commit -S -m "feat(nbd): adapter lifecycle, control-plane registration, bind gate"
```

---

