---
slug: /use_cases/iot
title: Internet of Things (IoT)
---

# Internet of Things (IoT)

**fluxrig** is a distributed runtime designed to solve the critical challenges of asynchronous data acquisition across unreliable networks. It functions as an **IoT Edge Runtime**, multiplexing what reaches it into a unified, cloud-native stream. Radio fleets on LoRaWAN, Zigbee, BLE or CAN bus arrive through the network server or concentrator that already terminates them, over TCP, HTTP or WebSocket; fluxrig sits behind that link rather than speaking the radio itself.

By shifting complexity to the **Distributed Node**, **fluxrig** allows organizations to reduce recurring carrier costs, maintain data integrity during backhaul outages, and retain full ownership of their telematic logic.

---

## Deployment patterns

Unlike monolithic IIoT gateways, **fluxrig** provides a flexible spectrum of operational patterns for high-density device orchestration.

### The autonomous field gateway
Ideal for disconnected environments (e.g., Cold Chain, AgTech), where the **Rack** acts as a high-reliability persistence node with deterministic finality.

*   **Deterministic Persistence**: Implements a high-concurrency **CBOR-encoded Binary WAL** (Write-Ahead Log) so buffered data survives LTE or Satellite network dropouts. (At-rest encryption of the WAL is a **[Roadmap]** feature; until it ships, do not persist regulated data unencrypted.)
*   **Edge Data Filtering**: Analyzing data locally at the source. By only transmitting "Significant Events" (e.g., Temperature Variance > 0.5C) and discarding redundant heartbeats, deployments can substantially cut recurring carrier costs.

### The transparent asset proxy
A deployment model for individual telematic units or industrial vehicles.

*   **Sovereign Telemetry**: Use the **[Bento Gear](../reference/gears/bento.md)** to map what the gateway publishes over TCP, HTTP or WebSocket into semantic `fluxMsg` events. A CAN or Modbus segment reaches it through the concentrator that already terminates it.
*   **Zero Trust Identity**: Every asset operates with independent mTLS certificates, ensuring that a compromised peripheral cannot impact the wider fleet.

### The LPWAN consolidation hub
Multiplex a dense sensor estate, whatever radio it runs on, into one stream: the network server terminates the radio and a Rack takes it from there.

*   **Telemetry Aggregation**: The hub collects data from thousands of low-power leaf nodes, flattens the telemetry into a unified schema, and flushes it to the Management Mixer via a secure mTLS tunnel.

---

## Visualization: The field gateway flow

<LikeC4 project="iot-gateway" view="flow" height={440} />

### Step-by-step processing
1.  **Data Multiplexing (1-2)**: Heterogeneous data from mesh and trackers are ingested into the Rack.
2.  **Deterministic WAL (3)**: Every bit is anchored to the local immutable archive before any network processing occurs.
3.  **Autonomous Filtering (4-5)**: Redundant data is discarded locally; only high-value events enter the encrypted tunnel.
4.  **Sovereign Command (6)**: The central Mixer receives a clean, normalized, and secured stream for orchestration.

---

## Institutional permissive freedom

**fluxrig** is a platform for institutional builders provided under the **Apache 2.0** license.

The licence does not change with the size of a deployment, and the normalization
logic is defined locally rather than in a provider's console:

*   **The same terms at any scale**: ten nodes or ten thousand are the same licence.
*   **A sovereign data layer**: the logic that normalizes your field data lives in your scenarios, so moving between cloud providers does not mean rewriting your field infrastructure.

> [!TIP]
> **Verification Strategy**: For large-scale fleet deployments, use **Scenario-Driven Simulation** to verify that your Filtering Logic maintains data integrity across simulated network partitions.

---

## Implementation reference

| Gear | Function | Status |
| :--- | :--- | :--- |
| **[Bento](../reference/gears/bento.md)** | Universal Protocol Bridge | **Stable** (MQTT and CAN need a custom build; see the gear reference) |
| **[io_lorawan](../reference/gears/io_lorawan.md)** | Native LoRaWAN LNS Integration | **Planned** |
| **[Wasm Logic](../reference/gears/wasm_logic.md)** | Custom Parsing & Edge Filtering | **Stable** |
