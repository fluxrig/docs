---
slug: /reference/protocol/wire
title: Wire protocol (snake)
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Wire protocol (snake)

This document serves as the canonical reference for the internal communication protocols, NATS subject topology, and message specifications that the **fluxrig** ecosystem uses.

---

## Snake protocol (transport layer)

The **Snake Protocol** is the secure mTLS transport layer. It connects distributed Racks to the central Mixer.

*   **Transport**: NATS TCP + TLS.
*   **Security**: Mutual TLS (mTLS).
*   **Encoding**: Binary (CBOR).

---

## NATS subject topology

**fluxrig** uses a hierarchical subject space (JetStream) to segregate traffic types.

| Scope | Pattern | NATS Strategy | Role |
| :--- | :--- | :--- | :--- |
| **Enrollment**| `flux.agent.>` | `Core NATS` | Handshakes, Heartbeats, and Adoption flows. |
| **Control** | `flux.rack.>` | `WorkQueue` | Targeted scenario updates and remote commands (Shutdown, Log-Level). |
| **Data** | `flux.msg.>` | `Stream` | High-volume `fluxMsg` traffic (The Hot Path). |
| **Telemetry**| `flux.telemetry.>` | `Stream` | System events (Audit logs, Status changes, Metrics). |

### Detailed subject structure

#### Enrollment & heartbeats
*   `flux.agent.hello`: New Racks broadcast it for initial enrollment.
*   `flux.agent.heartbeat`: Active Racks send it as periodic status reports.
*   `flux.agent.notify.{entity_id}`: The Mixer sends targeted commands through it (Adoption, Reconnect).

#### Control plane
*   `flux.rack.{rack_name}.scenario`: The Mixer pushes scenario updates to a specific Rack through it.

#### Data plane (hot path)
*   `flux.msg.{source_rack}.{gear_name}.{port}`: This pattern carries inter-gear communication as a stream.
*   Example: `flux.msg.rack-alpha.iso-server.out`

### Quality of service (QoS) & prioritization
To ensure the resilience of the platform during high-load scenarios, **fluxrig** implements strict QoS separation:

*   **Business Traffic (`flux.msg.>`):** It has highest priority. The system provisions NATS memory limits and JetStream buffers so transactional (`fluxMsg`) data arrives ahead of telemetry. This keeps business-path latency low under load.
*   **Telemetry Traffic (`flux.telemetry.>`):** The Rack queues it locally (embedded tier) and rate-limits it. If uplink bandwidth runs short, the system throttles telemetry ingestion (Token Bucket) to prevent bufferbloat from stalling primary business operations.

---

## Internal bus (NATS)
The central nervous system of **fluxrig** is NATS JetStream. All internal components communicate by emitting and consuming messages from specifically patterned subjects.

See **[Data Model (fluxMsg)](data_model.md)** for details of the message structure and field dictionary.