---
slug: /overview/foundation
title: Foundation components
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Foundation components


**fluxrig** moves protocol traffic between systems that do not share a format, a
transport, or an owner. A scenario describes what each node does.
A change to a flow is therefore a configuration change.

## Core pillars

1.  **Deterministic data plane**: every message uses **Deterministic CBOR (RFC 8949)** for serialization. The same message therefore produces the same bytes on every node. An audit trail can be compared rather than trusted.
2.  **Distributed edge**: The local processing node (Rack) holds its identity and last scenario in local state. It reconnects to the Mixer by itself after a connectivity failure. It keeps running the flows that stay inside it. It can start without the Mixer.
3.  **Unified control plane**: one Mixer holds the registry of enrolled Racks. It deploys scenarios to them. It aggregates their telemetry.
4.  **Remote fleet management**: **[Planned]** Future releases will introduce native Over-the-Air (OTA) update capabilities for Racks and Gears. They allow secure, remote lifecycle management of distributed infrastructure.

## Design philosophy

The platform is technically a distributed system. Its architecture follows the principles of modularity and high-integrity data flow.

> **[Read more about the professional audio logic →](./philosophy.md)**

## Strategic alignment

*   **Industrial (UNS)**: Implements the **Unified Namespace (UNS)** and **Industrial DataOps** principles to transform fragmented OT silos into a contextualized, event-driven hierarchy.
*   **Payments (PCI-DSS)**: Uses the **Stateless Context (Coat Check)** pattern to park correlation context at the edge. This keeps the primary processing path lean while the reply matches back deterministically.
*   **DevOps & SRE**: Delivers **Zero-Agent Observability** (OpenTelemetry) and **Configuration-as-Code** deployments, reducing operational overhead and firewall complexity.

---

## The nervous system (NATS)

The backbone of **fluxrig** is a resilient, distributed messaging mesh that provides the connective tissue for the entire platform.

*   **Goal**: Connect distributed racks securely with **guaranteed delivery**.
*   **The tool**: **[NATS JetStream](https://nats.io)**.
*   **Why**:
    *   **Resilient mesh**: Unlike traditional load balancers, NATS creates a pervasive mesh that handles connectivity gaps automatically.
    *   **Reconnection**: A Rack's NATS client reconnects by itself when the Mixer returns. NATS leaf nodes at the edge, which would keep guaranteed delivery inside a Rack while the Mixer is away, are `[Roadmap]`.
    *   **Financial-grade reliability**: We use JetStream to ensure **at-least-once** delivery and **strict ordering**, critical for financial events and command protocols.
    *   **Deterministic Data Encoding**: Low-bandwidth serialization via **[CBOR (RFC 8949)](https://cbor.io)** ensures data integrity and responsiveness over satellite or cellular links.

### Stateless context (the coat check)

To maintain performance and compliance (like PCI-DSS), we avoid bloating messages with heavy state. Instead, we use a "Coat Check" pattern. Correlation context parks in a ticket store (in-process by default, or a shared store when any instance must redeem the reply). It re-attaches unchanged when the reply returns.

> [!TIP]
> This pattern keeps the primary processing path lean. It **parks and restores** context. It does not tokenize (surrogate substitution is a separate, roadmap gear). It is not at-rest encryption (use in-memory storage, run hardened, for regulated data). See the **[Message Flow](../architecture/message_flow.md)** for technical details.

## The dual-pipeline strategy

We distinguish sharply between **Business Data** (Transactions) and **Operational Data** (Metrics/Logs). This requires two distinct pipeline patterns.

### Pipeline A: business data pipeline (gears & Wasm)
*   **Goal**: parse and transform protocol messages, from ISO 8583 financial traffic to sensor payloads.
*   **The strategy**: An extensible architecture designed for both performance and custom plugins.
    *   **Native gears**: Optimized Go logic for latency-sensitive protocols like **ISO8583** and financial switching.
    *   **WebAssembly (Wasm)**: Sandboxed, polyglot plugins (Rust, C++, TypeScript), allowing for safe third-party extensions.
    *   **Bento integration**: A rich ecosystem of standard cloud connectors (File, Stdout, HTTP) and powerful mapping (**bloblang**).
    *   **Side-Chain Inference**: **[Roadmap]** Asynchronous AI logic for non-deterministic tasks like fraud scoring or synthetic test generation.

### Pipeline B: operational data pipeline (OpenTelemetry)
*   **Goal**: Monitor the health of the system and trace transactions.
*   **The tool**: **[OpenTelemetry](https://opentelemetry.io)**.
*   **Strategy**: **Non-Intrusive Telemetry Tapping**.
    *   **Embedded library**: We use the **OTel Go SDK** directly inside the Rack binary. No sidecars or external agents required.
    *   **Telemetry tapping**: The Rack collects telemetry directly from the processing path. It diverts telemetry for monitoring without impacting the integrity of the primary data flow.
    *   **Secure transport**: The Rack serializes metrics and traces. It pushes them over a secure management tunnel to the central Mixer.
    *   **Aggregation**: The Mixer collects these streams. It forwards them to the storage layer for deep analytics.

## Deep storage (data sovereignty)

*   **The philosophy**: "Your data is yours".
*   **The solution**: Data is persisted in open standards, ensuring an **Immutable Audit Trail**.
    *   **Standard tier**: Zero-dependency local storage using **Parquet** and **DuckDB**.
    *   **Enterprise tier**: **[Roadmap]** Distributed high-availability storage with support for external backends (e.g., ClickHouse or S3-backed data lakes).

*   **The benefit**: Query your audit logs with any modern tool (Spark, pandas, Athena) without vendor lock-in.

---

## Summary: the stack

| Component | Role | Execution Model |
| :--- | :--- | :--- |
| **fluxrig Rack** | The Engine | Main Process |
| **NATS** | Messaging | Embedded Mesh Node |
| **CBOR** | Data Format | Deterministic Binary Encoding |
| **OpenTelemetry** | Observability | Compiled into Binary (Go SDK) |
| **Bento** | Cloud Connectors | Compiled into Binary (Go Module) |
| **ClickHouse** / **DuckDB** | Deep Storage | External DB / Embedded DB |
| **ARC** | Unified Reporting | External Enterprise Layer [Roadmap] |
| **Temporal** | Durable Workflow | External Service [Roadmap] |
| **Wasm** | Sandboxed Logic | Polyglot Plugins |
