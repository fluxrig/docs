---
slug: /development/standards
title: Engineering standards
---

# Technical engineering standards

This document defines the technical standards and **Technical Invariants** for the **fluxrig** ecosystem. Adherence to these rules ensures the platform remains deterministic, secure, and maintainable across disparate edge environments.

## Go idioms and style

We follow standard Go conventions (Effective Go) with strict enforcement via `golangci-lint`.

### Formatting and imports
*   **Auto-format**: All code must represent the output of `gofmt`.
*   **Imports**: Organize imports into four groups separated by whitespace:
    1.  Standard Library.
    2.  Third-party libraries.
    3.  Internal generic packages (e.g., `pkg/idgen`).
    4.  Internal specific packages (e.g., `pkg/mixer`).

## Error handling and the trace standard
Error handling is an architectural component of system reliability, not an afterthought.

*   **The Contextual Wrap**: Always wrap errors using `%w`, so the chain of causes survives to the caller that reports it.
    - `return fmt.Errorf("failed to process signal %s: %w", signalID, err)`
*   **The Double-Log Policy**: Never log an error *and* return it to the same caller. Log at the architectural boundary or the highest possible consumer to avoid signal duplication.
*   **Structured Attributes**: When returning an error, ensure the error type itself is verifiable using `errors.Is` or `errors.As`.

## Concurrency and lifecycle
*   **Context propagation**: Pass `context.Context` as the first argument to all hardware, I/O, or long-running operations.
*   **Goroutine hygiene**: Never start a goroutine without a clear termination plan. Use `sync.WaitGroup` or `golang.org/x/sync/errgroup` to manage lifecycles.

## Design for testability
We prioritize **Interface-Driven Development** so a collaborator can be replaced by a test double.

- **Dependency Injection**: Never hardcode global singletons (e.g., a DB connection) inside a package. Pass interfaces via constructors (`New...` functions).
- **Mocks & Fakes**: Packages must provide internal "Fakes" for their primary interfaces to support unit testing in downstream modules without external dependencies.

## Observability and structured logging
Logging is an architectural component for signal auditing, not a debugging output.

*   **Slog**: All logging must use the standard `log/slog` package. Never use `fmt.Printf` or `log.Print` for operational events.
*   **Structured Context**: Mandatory attribution of the `flux_id`, `gear_id`, and `trace_id` as structured fields.
*   **Institutional Tracing**: We maintain a dedicated `logger.LevelTrace` for high-resolution diagnostic mapping of signal paths.

---

## Signal integrity (Deterministic CBOR)
To ensure the long-term integrity of the **fluxMsg** data plane, we enforce **[Deterministic CBOR (RFC 8949)](https://www.rfc-editor.org/rfc/rfc8949.html)**.

*   **Map Sorting**: All map keys must be sorted lexicographically by their encoded bytes.
*   **Binary Identicalité**: The same logical structure must results in an identical byte-for-byte signature. This is mandatory for the **Passport (`state.flux`)** to remain cryptographically stable.
*   **Unknown Field Persistence**: Gears must preserve and forward unknown fields to maintain architectural compatibility.

---

## Documentation-as-code

Documentation is an engineering artifact.

### Godoc standards
We follow the standard Godoc conventions. If a symbol is exported, it **must** have a comment.

- Comments must be full sentences starting with the symbol name.
- Describe the *intent* and any critical constraints (e.g., "Thread-safe", "Blocking").

### Technical diagrams
Use **Mermaid.js** for all architectural and message flow diagrams.

- **Standardize** on `graph LR` (Left to Right) for sequence and topology views.
- Use portable `A == Label ==> B` thick-arrow syntax for maximum rendering fidelity.

---

## Build invariants (CGo)

When building binaries manually or via CI, the following **Build Invariants** are strictly enforced to ensure cross-platform compatibility:

*   **The Rack / CLI (`fluxrig`)**: Pure Static Go. Must compile with **`CGO_ENABLED=0`** to ensure compatibility with **Distroless Scratch** images and various Linux distributions.
*   **The Mixer (`fluxrig-mixer`)**: Embedded Performance. Must compile with **`CGO_ENABLED=1`** to link the **DuckDB** C++ engine for embedded OLAP analytics.

---

## Technical references
To implement these standards effectively, cross-reference the following technical deep-dives:

- **[Mixer API Reference](../reference/api.mdx)**: REST definitions and lifecycle management.
- **[Wire Protocol (Snake)](../reference/protocols.md)**: Low-latency NATS transport layer specifics.
- **[Developing Specialized Gears](../tutorials/writing_gears.md)**: Authoritative SDK lifecycle.
