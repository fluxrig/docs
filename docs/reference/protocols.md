---
slug: /reference/protocol/wire
title: Wire protocol (snake)
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Wire protocol (snake)

This document serves as the canonical reference for the internal communication protocols, NATS subject topology, and message specifications used within the **fluxrig** ecosystem.

---

## Snake protocol (transport layer)

The **Snake Protocol** is the secure mTLS transport layer connecting distributed Racks to the central Mixer.

*   **Transport**: NATS TCP + TLS.
*   **Security**: Mutual TLS (mTLS).
*   **Encoding**: Binary (CBOR).

---

## NATS subject topology

**fluxrig** utilizes a hierarchical subject space (JetStream) to segregate traffic types.

| Scope | Pattern | NATS Strategy | Role |
| :--- | :--- | :--- | :--- |
| **Enrollment**| `flux.agent.>` | `Core NATS` | Handshakes, Heartbeats, and Adoption flows. |
| **Control** | `flux.rack.>` | `WorkQueue` | Targeted scenario updates and remote commands (Shutdown, Log-Level). |
| **Data** | `flux.msg.>` | `Stream` | High-volume `fluxMsg` traffic (The Hot Path). |
| **Telemetry**| `flux.telemetry.>` | `Stream` | System events (Audit logs, Status changes, Metrics). |

### Detailed subject structure

#### Enrollment & heartbeats
*   `flux.agent.hello`: Broadcast by new Racks for initial enrollment.
*   `flux.agent.heartbeat`: Periodic status reports from active Racks.
*   `flux.agent.notify.{entity_id}`: Targeted commands from Mixer (Adoption, Reconnect).

#### Control plane
*   `flux.rack.{rack_name}.scenario`: Pushed scenario updates for a specific Rack.

#### Data plane (hot path)
*   `flux.msg.{source_rack}.{gear_name}.{port}`: Stream pattern for inter-gear communication.
*   Example: `flux.msg.rack-alpha.iso-server.out`

### Quality of service (QoS) & prioritization
To ensure the resilience of the platform during high-load scenarios, **fluxrig** implements strict QoS separation:

*   **Business Traffic (`flux.msg.>`):** Given highest priority. NATS memory limits and JetStream buffers are provisioned so transactional (`fluxMsg`) data is delivered ahead of telemetry, keeping business-path latency low under load.
*   **Telemetry Traffic (`flux.telemetry.>`):** Locally queued (embedded tier) and rate-limited. If uplink bandwidth is constrained, telemetry ingestion is throttled (Token Bucket) to prevent bufferbloat from stalling primary business operations.

---

## Internal bus (NATS)
The central nervous system of **fluxrig** is NATS JetStream. All internal components communicate by emitting and consuming messages from specifically patterned subjects.

See **[Data Model (fluxMsg)](data_model.md)** for a deep dive into the message structure and field dictionary.