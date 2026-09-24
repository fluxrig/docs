---
slug: /
title: fluxrig
hide_title: true
---

# fluxrig

**fluxrig** is a connectivity and protocol orchestration platform for distributed mission-critical infrastructure.

It provides a unified control plane to route, transform, and monitor data streams
across heterogeneous environments, from industrial IoT sensors to global payment
networks. The routing and transformation logic is declarative configuration,
which holds as much for a flow being designed now as for one that has been in
production for a decade.

A **Rack** is an edge node running a **scenario**: a YAML description of
processing steps and how they are wired. The **Mixer** enrolls Racks, deploys
scenarios to them, and collects their telemetry. A Rack keeps its identity and its
last scenario in local state, and reconnects to the Mixer by itself after an outage.

> [!TIP]
> Start with the **[5-Minute Quickstart](./tutorials/quickstart.md)**, or read the **[Architecture Overview](./architecture/overview.md)** and the **[use cases](./use_cases/index.md)**.

---

## Why fluxrig?

**fluxrig** is designed for environments where data integrity and edge resilience are non-negotiable.

1. **One control plane for many edge nodes**: Racks enroll themselves, receive scenarios, and report telemetry to a single Mixer.
2. **Protocol Agnostic**: A native ISO 8583 codec whose dialects are described by a spec rather than by code, generic TCP framing for binary protocols, and JSON and the rest through Bento mappings.
3. **Configuration over Code**: Deploy complex routing and transformation logic via declarative YAML: no custom coding required for standard integrations.
4. **Edge resilience**: A Rack keeps its identity and last scenario in local state, reconnects by itself after a backhaul outage, and keeps running the flows that stay inside it, and can start without the Mixer.

### Design philosophy: The professional audio logic

We borrow our architectural nomenclature from professional audio engineering to describe complex data flows with both precision and scale:

*   **The Gear**: A modular unit of logic (an ISO 8583 codec, a Bloblang mapping, a Wasm module of your own).
*   **The Rack**: An edge node that hosts and executes multiple Gears (like a stage rack).
*   **The Mixer**: The central Front of House (FOH) control plane that manages the fleet and aggregates telemetry.

> **[Read more about the professional audio logic →](./overview/philosophy.md)**

---

## Technical Architecture

```mermaid
flowchart LR
    subgraph Southbound ["The Edge (Southbound)"]
        Terminal[POS terminal]
        Device[Field device or gateway]
        subgraph Rack_A [**fluxrig** Rack]
            GearISO[ISO 8583 gear]
            GearIO[I/O gear]
        end
        Terminal -->|ISO 8583 over TCP| GearISO
        Device -->|TCP, HTTP| GearIO
    end
    subgraph Cloud ["Control Plane (Northbound)"]
        Mixer[fluxrig Mixer]
        Warehouse[\Parquet Archives\]
        subgraph Ops ["Management"]
            CLI[fluxrig CLI]
            Studio["Control Center <br/><i>(Planned)</i>"]
        end
    end
    Rack_A == "Secure Tunnel (mTLS)" ==> Mixer
    CLI --> Mixer
    Studio --> Mixer
    Mixer --> Warehouse
```

### Core Entities

| Entity | Technical Role | Analogy |
| :--- | :--- | :--- |
| **Mixer** | Central control plane and entity registry. | Mixing Console |
| **Rack** | Distributed edge node executing local logic. | Equipment Rack |
| **Gear** | Modular functional unit (Native or Wasm). | Effect Pedal |
| **Wire** | Persistent, ordered message queue (NATS). | Signal Path |
| **Snake** | Secure, multiplexed mTLS tunnel. | Audio Snake |
| **fluxMsg** | Standardized internal data structure (CBOR). | High-Fi Signal |

---

## Choose your journey

### For Operators & SREs
**Maintain maximum uptime** and operational visibility. **fluxrig** provides the tools to orchestrate distributed racks, manage immutable snapshots, and analyze telemetry in real time.

*   **[Deployment architecture & binaries →](./architecture/deployment.md)**
*   **[Operations & CLI reference →](./reference/cli.md)**
*   **[Technical configuration (Mixer/Rack) →](./reference/configuration.md)**
*   **[Telemetry & analytics guide →](./reference/telemetry_analytics.md)**

### For Developers & SDK Users
**Build specialized processing logic** with minimal overhead. Leverage the Native Go SDK to create custom protocol drivers or use the declarative Bento Gear for high-speed message transformation.

*   **[Writing your first Native Gear →](./tutorials/writing_gears.md)**
*   **[SDK reference & contract →](./reference/sdk.md)**
*   **[Bento Gear declarative logic →](./reference/gears/bento.md)**
*   **[Gears ecosystem reference →](./reference/gears/index.md)**

### For Architects & Security Teams
**Design resilient, sovereign data planes.** **fluxrig** allows for the design of complex, multi-actor topologies that enforce strict data isolation, zero-trust connectivity, and deterministic execution.

*   **[System foundation & core principles →](./overview/foundation.md)**
*   **[Security & identity architecture →](./architecture/security.md)**
*   **[Network & message flow integrity →](./architecture/message_flow.md)**
*   **[Industry use case spectrums →](./use_cases/index.md)**

---

<div class="pdf-center">
  <a href="https://fluxrig.org">fluxrig</a> is made with ❤️ in Uruguay 🇺🇾 by <a href="https://jaab.tech">JAAB Tech</a>
  <br /><br />
  <a href="https://jaab.tech" style={ { display: 'block', margin: '20px auto', textAlign: 'center' } }>
    <div class="flux-logo-small">
      ![JAAB Tech](assets/jaab_logo.svg)
    </div>
  </a>
</div>