---
title: Message flow and integrity
slug: /architecture/message-flow
---

# Message flow and integrity

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->



## Parallel processing and concurrency

While **fluxrig** maintains a logical sequence for each message path, the execution engine is built for high-density, parallel processing.

### The Goroutine-per-Wire model
Unlike legacy brokers that rely on single-threaded event loops or forked processes, the **Rack** leverages Go's native concurrency (Goroutines) to achieve high vertical scale on a single node.

*   **Isolated Execution**: Every defined **Wire** (subscription) in a Scenario is assigned its own goroutine.
*   **Automatic Parallelism**: If a Rack is deployed on a multi-core system, Gears across different wires execute in parallel automatically. This ensures that a high-latency I/O operation on one Gear (e.g., a slow database write) does not block the "Fast Lane" traffic on another.
*   **Zero-Sharding Overhead**: You do not need to manually shard processes or manage thread pools. The Rack runtime handles the M:N scheduling of thousands of concurrent signal paths with minimal memory overhead (~2KB per goroutine).

### Concurrency safety
Because Gears run in parallel, **fluxrig** enforces a strict separation of concerns:
*   **Stateless Gears**: Can process millions of messages concurrently without locks.
*   **Stateful Gears**: (e.g., ISO8583 Client or Coat Check) utilize internal synchronization primitives (Mutexes/Channels) to manage shared state safely while the platform continues to route other traffic.

---

---

## Wire directionality

Every **Wire** is **unidirectional**: it moves messages from exactly one output port to exactly one input port, in one direction only. There are no bidirectional wires, and a "round trip" through the system is always composed of separate one-way wires. Three rules follow:

1.  **Fan-out broadcasts.** Wiring one output port to several input ports duplicates every message to all of them. Fan-out is therefore a deliberate multicast, never a load-balancing mechanism: a gear that must choose *one* of N destinations (routing, pooling) needs one named output port per destination.
2.  **Fan-in merges.** Any number of wires may feed one input port; their streams merge safely. A gear therefore needs only one input port per *role* of incoming message (for example, "requests to route" vs "replies to match"), regardless of how many sources produce them.
3.  **Bidirectionality ends at the I/O boundary.** External connections (TCP/TLS sockets) are bidirectional, but they exist only at the outer edge of I/O gears: one socket maps onto two unidirectional ports (received bytes → `out`; `in` → written bytes). A request/response exchange with an external endpoint uses **both ports of the same I/O gear** and a pair of wires. See [the port model](./gear.md#the-port-model) and the [ISO8583 I/O gear](../reference/gears/io_iso8583.md#architectural-signal-path) for the canonical diagrams.

---

## Wire strategies

Not all data requires the same durability profile. **fluxrig** allows you to optimize the **Wire** per-flow based on the performance and durability requirements.

| Strategy | Transport | Durability | Status | Industry Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Standard** | **NATS JetStream** | Persisted before it is acknowledged | Shipped | High-assurance payments, auditable IoT, finality. |
| **Turbo** | **Go channels** | Volatile, lost on restart | **Planned** | Intranode logic that does not need to survive a crash. |

The trade is durability, not a number. What a hop costs depends on the machine,
the disk and the scenario, so it is measured rather than quoted: the
[roaming tutorial](../tutorials/roaming_enrichment.md) publishes a round trip
observed on a stated machine, with what it does and does not mean.

> [!WARNING]
> **Turbo Wires (Planned)**: Turbo Wires offer sub-millisecond performance by bypassing the NATS bus for local intra-rack flows. This strategy is currently in technical design and targeted for the **future milestone**.

---

## State management: metadata vs. coat check

To solve the context loss problem, **fluxrig** utilizes two distinct patterns depending on whether the data is within the trusted system mesh or crossing an external boundary.

### In-band Metadata (Intra-System Context)
When a message moves between Gears or Racks, it carries its context in-band via the **Metadata** map.

*   **Mechanism**: Key-value pairs stored directly in the `fluxMsg` envelope using [Deterministic CBOR (RFC 8949)](https://www.rfc-editor.org/rfc/rfc8949.html).
*   **Propagation**: The metadata travels with the message. When a Rack publishes to a durable stream, the entire envelope is persisted as a single atom of truth.
*   **Durability**: Guaranteed by the underlying transport layer with `At-Least-Once` delivery and high-availability retention.

### The Coat Check (Stateless Correlation)
When a message must leave the system to traverse an external network (e.g., raw TCP) that does not support the `fluxMsg` envelope, we implement the **Coat Check** pattern.

*   **The Drop-off**: Before the request exits the Rack, its metadata context is serialized and parked in a **ticket store**. The store is pluggable: `memory` (the default, in-process, RAM-only) for connection-bound flows, or a shared **NATS KV** store when any instance must redeem the reply. Correlation state is local to the Rack by default, not cluster-wide.
*   **The Ticket**: A unique identifier guaranteed to be returned by the external system (such as a Transaction Stand-in (STAN) or Retrieval Reference Number (RRN)) serves as the correlation key.
*   **The Pickup**: When the response message arrives, the Rack uses the "Ticket" to retrieve and re-attach the **same** parked context to the new `fluxMsg`, restoring traceability.

> [!IMPORTANT]
> **Parking is not tokenization.** The Coat Check *parks* a value and restores the **same** value on the reply, it keeps correlation context (and, optionally, specific fields) off a leg, but it does not substitute a surrogate. **Tokenization** (replacing a PAN with a surrogate backed by a persistent vault) is a distinct, separate gear on the roadmap. Note also that a payment switch must send the PAN to the scheme to authorize, so the PAN is not parked on the primary path.

> [!NOTE]
> The **[Conductor gear](../reference/gears/conductor.md)** generalizes this pattern: it adds connection routing and reply correlation over a "valet" engine (the ticket store above), and supersedes the standalone Coat Check for switching.

---

## The context lifecycle

The following sequence illustrates the **Coat Check** pattern during a typical asynchronous transaction excursion.

```mermaid
sequenceDiagram
    participant POS as External Device
    participant Rx as Rack
    participant KV as Ticket store (valet)
    participant Bank as External Processor
    
    POS->>Rx: Request A (STAN: 1234, PAN: ***)
    Rx->>Rx: Generate Metadata (Context, ID)
    Note over Rx,KV: COAT CHECK: Drop-off
    Rx->>KV: Put(Key=1234, Val=Metadata)
    Rx->>Bank: Send Raw Data (No Context)
    
    Note over Rx,Bank: ... Asynchronous Excursion ...
    
    Bank->>Rx: Response B (STAN: 1234)
    Note over Rx,KV: COAT CHECK: Pickup
    Rx->>KV: Get(Key=1234)
    KV-->>Rx: Return Metadata
    Rx->>Rx: Merge Metadata into fluxMsg B
    Rx->>POS: Authorized Response
```

---

## Control plane signaling

Beyond business data, **fluxrig** maintains a dedicated, high-priority **Control Plane Signaling** hierarchy (`flux.ctrl.>`) for out-of-band management and safety triggers.

### Message types & patterns

| Signal | Subject Pattern | Description | Impact |
| :--- | :--- | :--- | :--- |
| **Kill Switch** | `flux.ctrl.kill.>` | Emergency cessation of Gear processing. | Immediately halts the target Gear's internal loops. |
| **Conn Close** | `flux.ctrl.close.>` | Orchestrated termination of an I/O transport. | Triggers a clean socket closure and resource release. |
| **Scenario Update** | `flux.ctrl.sync.>` | Pushing a new execution topology. | Initiates the [Hot-Reload Process](deployment.md#operational-lifecycle-hot-reload). |

### Security & delivery
*   **Order of Precedence**: Control messages always bypass the standard data-plane queues to ensure immediate execution, even if the primary business queues are saturated.

---

## Reliability: connectivity convergence
To keep the first messages from being lost while subjects propagate, **fluxrig** implements a relentless connectivity handshake during every deployment and hot-reload.

### The Relentless Handshake
When a Rack starts or reloads a Scenario, it does not immediately activate the gear logic. Instead, it enters a **Convergence Phase**:

1.  **Sync Probes**: The Rack emits `FlagSyncProbe` messages (internal NATS control messages) across every defined Wire in the topology.
2.  **Propagation Loop**: These probes are re-emitted on `rack.handshake_interval`, which defaults to `500ms`, to cover JetStream propagation lag.
3.  **Finality Check**: The Rack waits until every path confirms it is "hot" and reachable across the distributed nodes.
4.  **Gear Activation**: Only after 100% convergence is confirmed are the business and protocol gears (e.g., ISO8583/Wasm) allowed to start processing real-world traffic.

> [!NOTE]
> This mechanism solves the **First-Message Loss** problem typically found in distributed messaging systems, where JetStream subjects may take milliseconds to propagate to all nodes after a topology change.

---

## Reliability: sagas and compensation messages

**fluxrig** treats failures as data rather than exceptions. This allows for the orchestration of complex, distributed transactions without fragile locks, prioritizing deterministic terminal states.

*   **Pattern: Optional Error Routing**: Gears *can* define a logical `.err` port for error handling. Note that this is a **Logic-Driven Pattern**: the engine provides the wiring infrastructure, but the individual Gear implementation must be coded to explicitly emit problematic data to the `.err` port upon failure.
*   **Saga Pattern**: This pattern enables the implementation of Sagas, where a failure at a specific node triggers a compensating message (e.g., a reversal or an automated alert) to restore the system to a clean terminal state.
*   **Finality Governance**: We enforce a policy where every message eventually reaches a "Success" or "Failure" state, ensuring the system remains self-healing, auditable, and compliant with institutional data standards.

> [!TIP]
> **Transport Abstraction**: By leveraging high-level messaging abstractions, **fluxrig** decouples business logic from the underlying NATS transport. This allows you to test complex Gear logic in-memory without a network server, ensuring technical validation during development.