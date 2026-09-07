---
slug: /use_cases/industrial
title: Industrial use cases
---

# Industrial use cases

**fluxrig** is a distributed runtime engineered for the convergence of Operational Technology (OT) and institutional IT infrastructure. It functions as an **Industrial DataOps Runtime**, providing a memory-safe execution layer for protocol normalization, heterogeneous data aggregation, and deterministic logic at the factory edge.

By composing specialized **Gears**, industrial engineers can transform what the shop floor already publishes into semantic, cloud-ready events without modifying mission-critical control hardware. Reading PLC registers directly needs the Modbus and RS-485 gears, which are on the roadmap; today a Rack sits behind whatever already speaks TCP or HTTP.

## The Unified Namespace (UNS) strategy
**fluxrig** is a native implementation of the **Unified Namespace** architectural pattern. By using the **Mixer** as the central source of truth for the entity registry and the secure mTLS transport, the platform enables a single, contextualized hierarchy for all industrial data, bridging the gap between the shop floor (OT) and the boardroom (IT).

> [!TIP]
> This strategy is central to modern **Industrial DataOps**. For more context on the terminology and industry alignment, see the **[Foundation Components](../overview/foundation.md)** guide.

---

## Operational patterns

Unlike monolithic IIoT gateways, **fluxrig** provides a flexible spectrum of operational patterns for industrial automation.

### Passive data acquisition (SPAN)
For legacy environments where touching the PLC cycle is restricted, **fluxrig** operates as a passive observer.

*   **Zero Impact**: Capturing OT traffic from a switch mirror port allows for full telemetry acquisition with absolute isolation from the industrial physical process.
*   **Shadow Verification**: Running a digital twin in parallel with legacy SCADA systems to validate new logic before a production cutover.

### Active normalization and control
In this pattern, **fluxrig** acts as an inline gateway, actively polling OT data and applying real-time logic before transmission to the central ecosystem.

*   **Protocol Aggregation**: Translation of what the fieldbus gateway already publishes into semantic `fluxMsg` events via the **[Bento Gear](../reference/gears/bento.md)**. Reading Modbus or Profinet registers directly needs the `[Roadmap]` **[Modbus gear](../reference/gears/io_modbus.md)**; today a Rack sits behind whatever terminates them.
*   **Autonomous Edge Filtering**: Executing high-frequency thresholding and filtering locally. Only relevant "Critical Events" (e.g., *Pressure Variance > 5%*) are transmitted, significantly reducing bandwidth and cloud ingestion costs.

---

## Visualization: The industrial verification rig

<LikeC4 project="industrial-rig" view="flow" height={420} />

### Step-by-step processing
1.  **Data Aggregation (1-2)**: Disparate data from PLCs and mesh sensors are merged into a unified internal bus.
2.  **Data Normalization (3)**: Data is normalized into the standard `fluxMsg` format, attaching sovereign metadata (trace_id, machine_id).
3.  **Audit Integrity (4)**: The **Correlator Gear** `[Roadmap]` ensures every industrial event is logged to the local immutable archive before external transmission.
4.  **Anomaly Detection (5)**: Real-time deviations are diverted to the SRE alerting tier for immediate operational response.

---

## Sovereign security: inbound zero

Industrial environments require absolute isolation. **fluxrig** enforces a strict **Inbound Zero** policy via secure mTLS tunnels:

*   **Outbound Initiation**: The localized Rack initiates the connection to the Central Mixer; no inbound ports are opened on the industrial firewall.
*   **Surface Attack Elimination**: By maintaining a closed-firewall profile, the factory floor remains invisible to public internet scanning and lateral movement attacks.

> [!CAUTION]
> **Industrial Warning: Execution Latency**
> 
> Running complex logic on resource-constrained distributed nodes can introduce execution latency.
>
> *   **Impact**: High CPU contention may delay high-frequency poller signals (e.g., sub-10ms RTU acquisition).
> *   **Recommendation**: Prioritize the **Stable [Bento Gear](../reference/gears/bento.md)** for data acquisition and maintain lean logic profiles to ensure deterministic data acquisition.

---

## Implementation reference

| Gear | Function | Status |
| :--- | :--- | :--- |
| **[Bento](../reference/gears/bento.md)** | Universal Protocol Bridge | **Stable** (Modbus, MQTT and SMTP need a custom build; see the gear reference) |
| **[io_modbus](../reference/gears/io_modbus.md)** | Native TCP/RTU High-Speed Poller | **Planned** |
| **[network_sniffer](../reference/gears/network_sniffer.md)** | Passive OT Traffic Capture | **Planned** |
| **[Wasm Logic](../reference/gears/wasm_logic.md)** | Custom Edge Filtering & Autonomy | **Stable** |
