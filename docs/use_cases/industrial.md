---
slug: /use_cases/industrial
title: Industrial use cases
---

# Industrial use cases

**fluxrig** is a distributed runtime for the convergence of Operational Technology (OT) and institutional IT infrastructure. It functions as an **Industrial DataOps Runtime**. It provides a memory-safe execution layer for protocol normalization, heterogeneous data aggregation, and deterministic logic at the factory edge.

By composing specialized **Gears**, industrial engineers can transform what the shop floor already publishes into semantic, cloud-ready events without modifying mission-critical control hardware. Reading PLC registers directly needs the Modbus and RS-485 gears, which are on the roadmap. Today a Rack sits behind whatever already speaks TCP or HTTP.

## The Unified Namespace (UNS) strategy
**fluxrig** is a native implementation of the **Unified Namespace** architectural pattern. The platform uses the **Mixer** as the central source of truth for the entity registry and the secure mTLS transport. This enables a single, contextualized hierarchy for all industrial data. This bridges the gap between the shop floor (OT) and the boardroom (IT).

> [!TIP]
> This strategy is central to modern **Industrial DataOps**. For more context on the terminology and industry alignment, see the **[Foundation Components](../overview/foundation.md)** guide.

---

## Operational patterns

Unlike monolithic IIoT gateways, **fluxrig** provides a flexible spectrum of operational patterns for industrial automation.

### Passive data acquisition (SPAN)
For legacy environments where you must not touch the PLC cycle, **fluxrig** operates as a passive observer.

*   **Zero impact**: Capturing OT traffic from a switch mirror port allows full telemetry acquisition with absolute isolation from the industrial physical process.
*   **Shadow verification**: Run a digital twin in parallel with legacy SCADA systems to validate new logic before a production cutover.

### Active normalization and control
In this pattern, **fluxrig** acts as an inline gateway. It polls OT data and applies real-time logic before transmission to the central ecosystem.

*   **Protocol aggregation**: Translate what the fieldbus gateway already publishes into semantic `fluxMsg` events via the **[Bento Gear](../reference/gears/bento.md)**. Reading Modbus or Profinet registers directly needs the `[Roadmap]` **[Modbus gear](../reference/gears/io_modbus.md)**. Today a Rack sits behind whatever terminates them.
*   **Autonomous edge filtering**: Execute high-frequency thresholding and filtering locally. The setup transmits only relevant "Critical Events" (e.g., *Pressure Variance > 5%*). This reduces bandwidth and cloud ingestion costs.

---

## Visualization: the industrial verification rig

<LikeC4 project="industrial-rig" view="flow" height={420} />

### Step-by-step processing
1.  **Data aggregation (1-2)**: Merge disparate data from PLCs and mesh sensors into a unified internal bus.
2.  **Data normalization (3)**: Normalize data into the standard `fluxMsg` format. Attach sovereign metadata (trace_id, machine_id).
3.  **Audit integrity (4)**: The **Correlator Gear** `[Roadmap]` logs every industrial event to the local immutable archive before external transmission.
4.  **Anomaly detection (5)**: Divert real-time deviations to the SRE alerting tier for immediate operational response.

---

## Sovereign security: inbound zero

Industrial environments require absolute isolation. **fluxrig** enforces a strict **Inbound Zero** policy via secure mTLS tunnels:

*   **Outbound initiation**: The localized Rack initiates the connection to the Central Mixer. The firewall opens no inbound ports.
*   **Surface attack elimination**: By maintaining a closed-firewall profile, the factory floor remains invisible to public internet scanning and lateral movement attacks.

> [!CAUTION]
> **Industrial warning: execution latency**
> 
> Running complex logic on resource-constrained distributed nodes can introduce execution latency.
>
> *   **Impact**: High CPU contention may delay high-frequency poller signals (e.g., sub-10ms RTU acquisition).
> *   **Recommendation**: Prioritize the **Stable [Bento Gear](../reference/gears/bento.md)** for data acquisition. Maintain lean logic profiles to ensure deterministic data acquisition.

---

## Implementation reference

| Gear | Function | Status |
| :--- | :--- | :--- |
| **[Bento](../reference/gears/bento.md)** | Universal Protocol Bridge | **Stable** (Modbus, MQTT and SMTP need a custom build. See the gear reference) |
| **[io_modbus](../reference/gears/io_modbus.md)** | Native TCP/RTU High-Speed Poller | **Planned** |
| **[network_sniffer](../reference/gears/network_sniffer.md)** | Passive OT Traffic Capture | **Planned** |
| **[Wasm Logic](../reference/gears/wasm_logic.md)** | Custom Edge Filtering & Autonomy | **Stable** |
