---
slug: /overview/philosophy
title: Design philosophy
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Design philosophy: the high-fidelity logic

While **fluxrig** is a technical orchestration engine, its architecture follows professional audio engineering principles. Managing distributed data flows is fundamentally identical to managing high-fidelity signals. This holds in a controlled **Recording Studio**. It holds on a stadium-scale **Live Event Stage**.

By adopting this nomenclature, we provide a consistent, intuitive mental model for complex distributed systems at any scale.

## Why the audio metaphor?

In professional audio, whether capturing a delicate acoustic performance or managing a stadium concert, signal integrity is the supreme mandate. Engineers must:

1.  **Isolate** processing logic into modular, swappable units.
2.  **Route** signals deterministically through persistent, failure-resistant paths.
3.  **Monitor** every point in the signal chain without interfering with the primary flow.
4.  **Orchestrate** at any scale (from the granular "patching" of microphones on a single stage to the global coordination of an entire world tour).

fluxrig applies these same uncompromising requirements to mission-critical business infrastructure.

### The studio vs. the stage

*   **The Studio Logic**: Focuses on **precision and fidelity**. Like a high-end signal chain, every transformation in fluxrig is deterministic and bit-perfect. This is the logic of the "Hot Path" (the microsecond-latency processing of individual transactions).
*   **The Stage Logic**: Focuses on **scale and orchestration**. A large-scale festival orchestrates all the instruments on a single stage (a **Rack**). It then connects multiple stages to a central console (The **Mixer**). In the same way, fluxrig manages thousands of edge nodes as a single, cohesive event.

## Core nomenclature

| Term | Technical definition | Studio analogy |
| :--- | :--- | :--- |
| **Mixer** | The centralized control plane and entity registry. | The **Mixing Console** or Front of House (FOH) station. |
| **Rack** | A distributed edge node that executes logic. | An **Equipment Rack** housing specialized signal processors. |
| **Gear** | A modular logic unit (protocol codec, adapter, or transformation). | A **Signal Processor**, pedal, or outboard effect unit. |
| **Wire** | A persistent, ordered data path (NATS JetStream). | A **Signal Path** or high-quality patch cable. |
| **Snake** | A secure, multiplexed mTLS tunnel for control signals. | A **Multi-core Snake** cable used for long-distance routing. |
| **fluxMsg** | The standardized data envelope (Deterministic CBOR). | A **Normalized Signal** calibrated for downstream processing. |

## The "studio" in practice

When you deploy **fluxrig**, you effectively build a distributed "recording studio" for your data:

-   **Modular logic (hot-swapping)**: Just as an engineer swaps a compressor pedal mid-session, you swap a **Gear**. You migrate from ISO8583 to JSON, for example, without re-architecting your entire signal chain.
-   **Zero-interference monitoring**: We "tap" the signal at the Gear level. This provides OpenTelemetry traces that mirror the data flow exactly without adding latency to the primary path.
-   **Two lanes**: A wire inside one Rack goes through memory. Nothing is stored and nothing goes to the Mixer. A wire between Racks, or one that asks for it, uses the guaranteed lane. NATS JetStream stores each message. It stores it before it tells the emitting gear that it accepted the message. A message emitted while the Rack cannot reach the bus on a guaranteed wire is not accepted. The emitting gear receives the error.

---

> [!TIP]
> While this metaphor provides architectural inspiration, the **[Technical Reference](../reference/index.md)** prioritizes standard industry terminology (Distributed Systems, IoT, Payment Orchestration) for precision and accessibility.
