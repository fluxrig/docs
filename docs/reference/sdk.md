---
slug: /reference/development/go-sdk
title: Native Go SDK contract
---

# Native Go SDK contract

The **fluxrig Native Go SDK** is the authoritative framework for building deterministic signal processing modules (Gears). It provides the primitives required to interact with the `fluxMsg` data stream and the sovereign control plane.

## The Gear interface

Every Native Gear must implement the standard `NativeGear` interface. This contract ensures that the Rack runtime can orchestrate initialization, signal processing, and graceful shutdown.

```go
type NativeGear interface {
    // Init loads configuration and prepares the gear.
    Init(ctx GearContext) error

    // Start begins the active lifecycle of the Gear.
    // 'emit' is used by Source Gears to inject signals into the Bus.
    Start(ctx context.Context, emit func(*fluxmsg.FluxMsg)) error

    // Process handles an incoming signal and returns a response signal (optional).
    Process(ctx context.Context, msg *fluxmsg.FluxMsg) (*fluxmsg.FluxMsg, error)

    // Drain signals the gear to stop accepting new input.
    Drain(ctx context.Context) error

    // Stop performs a final cleanup of Gear resources.
    Stop() error
}
```

### Key primitives

| Primitive | Purpose |
| :--- | :--- |
| **`GearContext`** | Provides access to the system Bus, Registry, and Telemetry Tap. |
| **`fluxMsg`** | The canonical, CBOR-ready data structure for all internal signals. |
| **`FluxID`** | Deterministic 128-bit UUID v7 identifiers for signal tracing. |

## Signal flow lifecycle

1.  **Ingestion**: A southbound Gear (e.g., `io_tcp`) consumes raw wire bytes and wraps them in a `fluxMsg` envelope.
2.  **Processing**: The signal is passed through a chain of Logic Gears (Native or Wasm) which modify the metadata or payload.
3.  **Outgestion**: A northbound Gear (e.g., `registry-sink`) persists the final signal or routes it to an external endpoint.

## Development workflow

Native Gears are compiled directly into the `fluxrig` binary for maximum performance. For third-party extensibility without recompilation, organizations should leverage the **[Wasm Logic Gear](../reference/gears/wasm_logic.md)**.

---

> [!NOTE]
> The complete Go source for the SDK can be found in the **[`pkg/sdk`](https://github.com/jaab-tech/fluxrig/tree/main/pkg/sdk)** directory. 
> 
> For implementation templates, see the **[`examples/`](https://github.com/jaab-tech/fluxrig/tree/main/examples)** folder in the official repository.
