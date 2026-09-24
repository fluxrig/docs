---
slug: /architecture/overview
title: Architectural overview
sidebar_label: System Design
---

# Architectural overview

The **fluxrig** architecture is a modular, event-driven system designed to separate centralized orchestration from distributed execution. It functions as a connectivity layer that enables deterministic processing across heterogeneous distributed environments.

> **Core Principle**: **Wires are Persistent Signal Paths** (NATS JetStream).
> Intra-Rack and Inter-Rack communication use the *same* protocol (`fluxMsg`), ensuring location transparency and uniform observability across the entire mesh.

## Design Philosophy

While the platform is technically a distributed system, its architecture is inspired by the modularity and signal integrity of professional audio studios. 

> **[Read more about the professional audio logic →](../overview/philosophy.md)**

## System components

The platform is built on a modular architectural model that strictly separates centralized orchestration from distributed execution.

| Component | Role | Description |
| :--- | :--- | :--- |
| **Mixer** | Control Plane | The central orchestration node and entity registry. Manages global state and configuration. |
| **Rack** | Distributed Node | A decentralized execution node running the processing logic (Gears) defined in a Scenario. |
| **Gear** | Logic Unit | A modular unit of processing logic (Native/Wasm) responsible for specific tasks. |
| **Wire** | Signal Path | A deterministic, unidirectional persistent queue (NATS JetStream) connecting gears. |
| **Snake** | Secure Tunnel | The persistent, outbound-only **mTLS** connection between a Rack and the Mixer. |
| **Scenario**| Topology Config | Declarative YAML defining the execution topology and signal routes. |
| **Spec** | Schema | Defines the structure of external payloads natively via **SDL**. |
| **Registry** | Identity Ledger | The centralized source of truth for node identity, security secrets, and telemetry indexing. |
| **fluxMsg** | Data Model | The universal binary data structure (CBOR) for all internal signals. |
| **Warehouse**| Storage | The Mixer flushes telemetry to Parquet beside DuckDB. Partitioning it as cold storage for audit logs is *[Roadmap]*. |
| **Studio** | Console UI | *[Roadmap]* The browser-based management console for visual topology building. |

---

## The gear model

A **Gear** is the primary logic unit. It functions as a pluggable, high-resolution signal processor that can be chained together via **Wires** to form complex, low-latency pipelines.

*   **Execution strategy**: **Native gears** are compiled into the binary; **Wasm gears** are sandboxed modules loaded at runtime, written in any language that targets WebAssembly.
*   **The Port Model**: Communication is strictly governed by standardized entry and exit points (**Ports**) that enforce signal integrity.
*   **Observability**: Every Gear is observable by default, with throughput and latency telemetry tapped directly from the execution path.

<LikeC4 project="concepts" view="pipeline" height={260} />

<br />

> [!NOTE]
> The pipeline above shows a **passthrough** flow: two distinct I/O gears bridge two different external connections, and every arrow is a unidirectional Wire. A **request/response** flow with a single external endpoint looks different: it uses **both ports of the same I/O gear**, with one wire pair carrying the request away from its `out` port and the response back to its `in` port. Only the external sockets are bidirectional; see [the port model](./gear.md#the-port-model).

> [!NOTE]
> For a deep dive into implementation models, port anatomy, and the Gear lifecycle, see the **[Architecture: Gear](./gear.md)** guide.

---

## The rack (distributed execution)

The Rack is the "execution case" running locally or at the edge. It is designed to be **Static, Self-Contained, and High-Performance**, acting as the runtime environment for local signal processing.

*   **Core**: A compiled **Go binary** (`fluxrig`).
*   **Deterministic Engine**: Embeds the Wasm runtime, and connects to the NATS JetStream server embedded in the Mixer.
*   **Zero-Agent Footprint**: Observability (OTel) and configuration management are embedded directly in the binary.
*   **Hosted Gears**: Executes gateways, codecs, and logic units as a unified pipeline.

### Recovery without intervention
A Rack starts from its Passport when the Mixer is unreachable and runs the last scenario it applied if none of its wires uses the bus, then joins the Mixer by itself when it returns, without stopping what it is running. A wire between two gears of one Rack goes through the Rack's memory and keeps running without the Mixer; what crosses between Racks travels over the bus embedded in the Mixer and waits for it. See [A Rack without the Mixer](../reference/operations.md#a-rack-without-the-mixer).

<LikeC4 project="rack" view="internals" height={440} />

---

## The control plane (Mixer)
The **Mixer** is the central nervous system and **Project Authority** for the rig. It governs identity, orchestrates topology updates, and aggregates telemetry.

*   **The Operational Ledger (Registry)**: A hybrid source of truth that maintains node identity, security secrets, and a distributed telemetry index. It leverages **DuckDB** for active state and **Parquet partitions** for historical analysis.
*   **Analytics**: A telemetry engine providing real-time operational visibility through the Ledger.
*   **Orchestrator**: Manages the deployment and hot-reloading of **Scenarios** across the fleet.

### The ledger architecture
The Registry functions as a **Unified Operational Ledger**. In the current release, it provides a single gateway to query both the **Active State** (currently connected nodes) and **Cold Storage** (historical logs and metrics partitioned by time).

- **SQL Access**: Direct querying of the active registry schema.
- **Unified Query**: Transparently joins active data with Parquet-backed historical trails using DuckDB's `read_parquet` extensions.

---

## The data plane (Rack)
The **Rack** is the execution engine that lives locally or at the infrastructure boundary. It is designed for absolute autonomy and wire-speed performance.

*   **Gear Runtime**: A lightweight, non-blocking execution environment.
*   **Local identity**: Racks start from their local **Passports** (cryptographically signed identity snapshots) without contacting the Control Plane. Flows that stay inside one Rack keep running while the Control Plane is away, and a Rack can start without it.
*   **OTel Ingest**: Local telemetry collection and buffering with resilient outbound delivery.

---

## The orchestration spectrum

Unlike monolithic gateways, **fluxrig** operates across an **Orchestration Spectrum**. We separate the "Hot Path" (Mission-Critical Signal Processing) from the "Cold Path" (High-Assurance Business Orchestration).

1.  **Passive Monitoring**: Non-intrusive shadow mirroring of traffic for risk-free validation.
2.  **Mission-Critical Signal Processing**: The low-latency Rack pipeline for protocol normalization and localized logic.
3.  **High-Assurance Business Orchestration**: The Mixer-hosted **Durable Orchestration** engine for long-running processes (e.g., settlements, disputes).

> [!TIP]
> For sector-specific implementations of this spectrum, see the **[Payment](../use_cases/payments.md)**, **[IoT](../use_cases/iot.md)**, and **[Industrial](../use_cases/industrial.md)** architectural spectrums.

---

## Transaction flow strategy (two-speed)

### The hot path: mission-critical signal processing
*   **Scope**: Real-time signal processing and authorization.
*   **Performance**: Gear latency is measured per message and exported; the suites in this repository report it. No figure is quoted here, because an overhead number without the machine and the load that produced it is not one.
*   **Engine**: **Native Rack pipeline**.

### The cold path: high-assurance business orchestration **[Roadmap]**
*   **Scope**: Long-running, stateful business processes (e.g., smart reversals, settlements).
*   **Goal**: Ensure eventual consistency and correctness for complex workflows.
*   **Engine**: **[Roadmap]** Durable Orchestration (Temporal).

### Workflow vs execution

| Feature | **Durable Orchestration** [Roadmap] | **Deterministic Processing** |
| :--- | :--- | :--- |
| **Role** | **Director**: Decides *what* and *when*. | **Processor**: Decides *how* (I/O & Logic). |
| **State** | **Stateful**: Full workflow history. | **Stateless**: Message-at-a-time. |
| **Latency** | **High** (ms to days). | **Ultra-Low**: (microseconds). |
| **Example** | **Example**: "Run Reconciliation A, then B." | "Transform ISO8583 to CBOR." |

---

## Security & data sovereignty
For detail on the **Snake Protocol (mTLS)** and **Air-Gapped Operation**, please refer to the dedicated **[Security Architecture](security.md)** and **[Data Architecture](data.md)** documents.