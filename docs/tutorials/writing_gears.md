---
slug: /tutorials/writing_gears
title: Developing specialized gears
---

# Developing specialized gears

**fluxrig** is designed as a modular orchestration engine where physical I/O and business logic are encapsulated as **Gears**. 

Before dropping into Go code, remember the **Orchestration Spectrum**:

*   **Declarative Logic**: The majority of business logic patterns (including normalization, mapping, and alerting) are most efficiently implemented using the **[Bento Gear](../reference/gears/bento.md)**. This declarative approach significantly reduces development overhead and long-term maintenance complexity.
*   **Polyglot Business Rules**: For secure, sandboxed execution of custom business logic in languages like Rust, Zig, or AssemblyScript, use the **[Wasm Logic Gear](../reference/gears/wasm_logic.md)**. See the **[Building Wasm Gears in Zig](./building_wasm_gears_zig.md)** tutorial.
*   **Specialized Protocols**: For protocol drivers (ISO 8583, Modbus), binary packers, or customized network stacks, you develop **Native Go Gears**.

## Prerequisites: The gear contract

A **Gear** is a modular plugin that satisfies the `sdk.NativeGear` interface. Unlike generic plugins, native Gears have direct access to the services of the hosting **Rack**, including structured logging, deterministic ID generation, and the OpenTelemetry signal path.

The lifecycle of a Gear follows a strict state machine to ensure zero-loss operations:

1.  **Init**: Strategic configuration mapping and validation.
2.  **Start**: Active lifecycle ignition (spawning listeners or dialers).
3.  **Process**: Sequential request/response transformation.
4.  **Drain**: Graceful input suppression (preparing for shutdown).
5.  **Stop**: Absolute resource reclamation.

To guarantee technical fidelity, the following tutorial uses production-grade code derived from the **Simple TCP Gear** (`pkg/gears/native/io_tcp`).

---

## Step 1: The architecture contract

Production Gears in `fluxrig` separate concerns across multiple files: `config.go` for settings, `gear.go` for the interface, and implementation files (e.g., `server.go`).

Every native component must satisfy the `sdk.NativeGear` interface:

```go
// pkg/gears/native/io_tcp/gear.go
package io_tcp

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/jaab-tech/fluxrig/pkg/fluxmsg"
	"github.com/jaab-tech/fluxrig/pkg/sdk"
)

// Gear is the outer wrapper that conforms to sdk.NativeGear
type Gear struct {
	config *Config
	log    *slog.Logger
	ctx    sdk.GearContext
	emit   func(*fluxmsg.FluxMsg)

	impl ModeImpl // Delegated to server or client logic
}

// ModeImpl is whatever this gear delegates its lifecycle to. Splitting it lets
// one gear serve two directions without branching in every method.
type ModeImpl interface {
	Start(ctx context.Context) error
	Drain(ctx context.Context) error
	Stop() error
}

// Ensure interface compliance at compile time
var _ sdk.NativeGear = (*Gear)(nil)
```

> [!TIP]
> **Compilation Safety**: The `var _ sdk.NativeGear = (*Gear)(nil)` declaration forces the compiler to verify implementation completeness during development.

---

## Step 2: Strategic initialization

The `Init` method is executed during the Rack's boot sequence. This is where raw configuration is transformed into a validated technical state.

```go
// Init loads configuration and prepares the gear.
func (g *Gear) Init(ctx sdk.GearContext) error {
	g.ctx = ctx
	g.log = ctx.Logger()

	//  Manual mapping from generic YAML/TOML
	cfg, err := ParseConfig(ctx.Config())
	if err != nil {
		return fmt.Errorf("io_tcp: invalid config: %w", err)
	}

	//  Hardened validation
	if err := cfg.Validate(); err != nil {
		return fmt.Errorf("io_tcp: validation failed: %w", err)
	}
	g.config = cfg

	g.log.Info("initialized", "mode", cfg.Mode, "bind", cfg.Bind)
	return nil
}
```

---

## Step 3: Active lifecycle ignition

The `Start` method begins the Gear's operational life. It provides the `emit` callback, allowing the Gear to inject signals into the signal path.

```go
func (g *Gear) Start(ctx context.Context, emit func(*fluxmsg.FluxMsg)) error {
	g.log.Info("starting gear")
	g.emit = emit

	switch g.config.Mode {
	case "server":
		g.impl = NewServer(g.config, g.log, g.emit, g.ctx.IDGen())
	case "client":
		g.impl = NewClient(g.config, g.log, g.emit, g.ctx.IDGen())
	default:
		return fmt.Errorf("io_tcp: unknown mode %q", g.config.Mode)
	}

	return g.impl.Start(ctx)
}
```

---

## Step 4: Networking resilience (The Accept Loop)

Robust networking at the edge requires protection against resource exhaustion. Below is the production implementation of the TCP `acceptLoop`, featuring exponential backoff to prevent spin-loops during OS failure states.

```go
// pkg/gears/native/io_tcp/server.go
func (s *Server) acceptLoop() {
	backoff := 5 * time.Millisecond

	for {
		conn, err := s.listener.Accept()
		if err != nil {
			if isActive(s.done) {
				s.log.Error("accept failed", "error", err)
				//  Exponential backoff on Accept failure
				time.Sleep(backoff)
				if backoff < 1*time.Second {
					backoff *= 2
				}
			}
			continue
		}
		backoff = 5 * time.Millisecond //  Reset on success

		//  Connection limits (DoS Protection)
		if s.config.MaxConnections > 0 && s.activeConns.Load() >= int64(s.config.MaxConnections) {
			s.log.Warn("max connections reached, rejecting")
			_ = conn.Close()
			continue
		}

		s.activeConns.Add(1)
		go s.handleConn(conn)
	}
}
```

---

## Step 5: Signal ownership and metadata

In a high-concurrency runtime, memory safety is paramount. When reading from hardware, we must explicitly copy the buffer and attach traceable metadata.

```go
func (s *Server) handleConn(conn net.Conn) {
	// ... (Setup and ID generation)
	
	scanner := bufio.NewScanner(conn)
	for scanner.Scan() {
		data := scanner.Bytes()
		//  Absolute Payload Ownership (Copy-on-Write)
		payload := make([]byte, len(data))
		copy(payload, data) 

		msg := fluxmsg.New()
		msg.flux_id, _ = s.idGen.NextFluxID()
		msg.RawPayload = payload

		//  Contextual Signal Metadata
		msg.Metadata["conn.id"] = connID
		msg.Metadata["peer.ip"], msg.Metadata["peer.port"], _ = net.SplitHostPort(conn.RemoteAddr().String())
		
		s.emit(msg) //  Signal Injection
	}
}
```

---

## Step 6: Replying to the source

The `Process` method handles asynchronous signals returning from the pipeline. We use the **Signal Metadata** to route the response back to the correct physical socket.

```go
func (s *Server) Process(ctx context.Context, msg *fluxmsg.FluxMsg) (*fluxmsg.FluxMsg, error) {
	//  Resolve session from metadata
	connID, ok := msg.Metadata["conn.id"]
	if !ok {
		return nil, fmt.Errorf("missing conn.id")
	}

	val, ok := s.conns.Load(connID)
	if !ok {
		return nil, fmt.Errorf("session %s lost", connID)
	}
	conn := val.(*Connection)

	//  Signal egress
	if len(msg.RawPayload) > 0 {
		_, err := conn.conn.Write(msg.RawPayload)
		if err != nil {
			return nil, fmt.Errorf("write error: %w", err)
		}
	}

	return msg, nil
}
```

---

## Step 7: Shutting down

`Drain` and `Stop` are the other half of the contract, and the compile-time
check above fails without them.

```go
// Drain stops taking new work and lets what is already in flight finish. It is
// called before Stop and carries a deadline: returning early is fine, running
// past it is not.
func (g *Gear) Drain(ctx context.Context) error {
	g.log.Info("draining gear")
	if g.impl == nil {
		return nil
	}
	return g.impl.Drain(ctx)
}

// Stop releases everything the gear holds. It runs after Drain, and must be
// safe to call on a gear that never started: a Rack that fails during startup
// stops what it already built.
func (g *Gear) Stop() error {
	g.log.Info("stopping gear")
	if g.impl == nil {
		return nil
	}
	return g.impl.Stop()
}
```

Both must be idempotent. A gear that panics or blocks the second time it is
stopped turns an orderly shutdown into a hung process.

---

## Step 8: Technical verification

Every specialized Gear must carry **unit verification** that keeps the logic remains resilient across releases.

```go
// pkg/gears/native/my_gear/gear_test.go
func TestGear_Process(t *testing.T) {
	g := &Gear{}
	// Mock the institutional context
	ctx := sdk.NewMockGearContext(map[string]any{"threshold": 100})
	
	_ = g.Init(ctx)
	
	msg := fluxmsg.New()
	msg.RawPayload = []byte("SIGNAL_DATA")

	out, err := g.Process(context.Background(), msg)
	
	assert.NoError(t, err)
	assert.NotNil(t, out)
}
```

> [!IMPORTANT]
> **Zero-Loss Shutdown**: Always implement the `Drain` and `Stop` hooks. The `Drain` phase signals the logic to stop accepting new signals while finishing work-in-progress, ensuring a graceful institutional handover.

---

## Integration: Deployment to the rack

Once registered in the `Factory`, deploy your Gear by defining it in `fluxrig.toml`:

```toml
[gears.ingress_gateway]
spec = "io_tcp"
config = { mode = "server", bind = "0.0.0.0:8000" }

[bus.routes.ingress_gateway]
targets = ["logic_processor"]
```

If you see your signal pulses flowing through the **[Observability Dashboard](../architecture/observability.md)**, you have successfully deployed a production-grade protocol driver to the Sovereign Edge.