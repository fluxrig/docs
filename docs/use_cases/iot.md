---
slug: /use_cases/iot
title: Internet of Things (IoT)
---

# Internet of Things (IoT)

**fluxrig** is a distributed runtime for asynchronous data acquisition across unreliable networks. It functions as an **IoT Edge Runtime**. It multiplexes what reaches it into a unified, cloud-native stream. Radio fleets on LoRaWAN, Zigbee, BLE or CAN bus arrive through the network server or concentrator that already terminates them, over TCP, HTTP or WebSocket. fluxrig sits behind that link. It does not speak the radio itself.

Shifting complexity to the **Distributed Node** allows this. **fluxrig** allows organizations to reduce recurring carrier costs. It allows them to maintain data integrity during backhaul outages. It allows them to retain full ownership of their telematic logic.

---

## Deployment patterns

Unlike monolithic IIoT gateways, **fluxrig** provides a flexible spectrum of operational patterns for high-density device orchestration.

### The autonomous field gateway
Use it for disconnected environments (e.g., Cold Chain, AgTech). The **Rack** acts as a high-reliability persistence node with deterministic finality.

*   **Deterministic persistence**: The setup implements a high-concurrency **CBOR-encoded Binary WAL** (Write-Ahead Log). Buffered data survives LTE or Satellite network dropouts. (At-rest encryption of the WAL is a **[Roadmap]** feature. Until it ships, do not persist regulated data unencrypted.)
*   **Edge data filtering**: Analyze data locally at the source. Transmit only "Significant Events" (e.g., Temperature Variance > 0.5C). Discard redundant heartbeats. This cuts recurring carrier costs.

### The transparent asset proxy
Use it for individual telematic units or industrial vehicles.

*   **Sovereign telemetry**: Use the **[Bento Gear](../reference/gears/bento.md)** to map what the gateway publishes over TCP, HTTP or WebSocket into semantic `fluxMsg` events. A CAN or Modbus segment reaches it through the concentrator that already terminates it.
*   **Zero trust identity**: Every asset operates with independent mTLS certificates. A compromised peripheral cannot impact the wider fleet.

### The LPWAN consolidation hub
Multiplex a dense sensor estate, whatever radio it runs on, into one stream. The network server terminates the radio. A Rack takes it from there.

*   **Telemetry aggregation**: The hub collects data from thousands of low-power leaf nodes. It flattens the telemetry into a unified schema. It flushes it to the Management Mixer via a secure mTLS tunnel.

---

## Visualization: the field gateway flow

<LikeC4 project="iot-gateway" view="flow" height={440} />

### Step-by-step processing
1.  **Data multiplexing (1-2)**: Ingest heterogeneous data from mesh and trackers into the Rack.
2.  **Deterministic WAL (3)**: Anchor every bit to the local immutable archive before any network processing.
3.  **Autonomous filtering (4-5)**: Discard redundant data locally. Only high-value events enter the encrypted tunnel.
4.  **Sovereign command (6)**: The central Mixer receives a clean, normalized, and secured stream for orchestration.

---

## Institutional permissive freedom

**fluxrig** is a platform for institutional builders provided under the **Apache 2.0** license.

The licence does not change with the size of a deployment. Define the normalization logic locally rather than in a provider's console:

*   **The same terms at any scale**: ten nodes or ten thousand are the same licence.
*   **A sovereign data layer**: the logic that normalizes your field data lives in your scenarios. Moving between cloud providers does not mean rewriting your field infrastructure.

> [!TIP]
> **Verification strategy**: For large-scale fleet deployments, use **Scenario-Driven Simulation** to verify that your Filtering Logic maintains data integrity across simulated network partitions.

---

## Implementation reference

| Gear | Function | Status |
| :--- | :--- | :--- |
| **[Bento](../reference/gears/bento.md)** | Universal Protocol Bridge | **Stable** (MQTT and CAN need a custom build. See the gear reference) |
| **[io_lorawan](../reference/gears/io_lorawan.md)** | Native LoRaWAN LNS Integration | **Planned** |
| **[Wasm Logic](../reference/gears/wasm_logic.md)** | Custom Parsing & Edge Filtering | **Stable** |
