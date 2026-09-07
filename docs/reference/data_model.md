---
slug: /reference/protocol/data-model
title: Data model (fluxMsg)
---

# Data model (fluxMsg)

This section details the specific wire formats (CBOR), field dictionaries, and entity identification used in the **fluxrig** ecosystem.

**Future Roadmap (future releases)**: We are currently re-implementing the Registry to leverage **[NATS KV](https://docs.nats.io/nats-concepts/jetstream/key-value-store) [Roadmap]** for all state governance. This will move the architecture from a "Push" model to a "Distributed State Watcher" model, increasing resilience and simplifying the handling of concurrent updates.

> [!NOTE]
> For the conceptual architecture and philosophy behind these models, see the **[Architecture: Data Model](../architecture/data.md)** guide.

## Message specification (header spec)

All messages traversing the bus MUST adhere to the **Signal Metadata** standard for context propagation.

### Wire headers (transport level)

These headers are used during message propagation across the NATS network or HTTP boundaries. *Note: These are distinct from the internal `Metadata` map found within the payload body.*

| Header Key | Standard | Description |
| :--- | :--- | :--- |
| `Nats-Msg-Id` | NATS | Unique Message Deduplication ID. |
| `Traceparent` | W3C OTel | Distributed Tracing ID (`00-{trace_id}-{span_id}-{flags}`). |

### fluxMsg structure (payload)

The payload is always a CBOR-encoded `fluxMsg` struct.

```go
// fluxMsg is the atomic unit of data in the system.
type FluxMsg struct {
    // --- Identity & Tracing ---
    FluxID    uuid.UUID `cbor:"id"`      // Global Unique ID (UUID v7)
    RefFluxID uuid.UUID `cbor:"ref_id"`  // Correlation / Parent ID
    TraceID   string    `cbor:"trace"`   // OpenTelemetry trace_id (Hex)
    SrcGearID uuid.UUID `cbor:"src_id"`  // Originator EntityID

    // --- Context (Metadata) ---
    // Routing flags, source IP, protocol headers.
    // Values MUST be valid UTF-8 strings.
    Metadata map[string]string `cbor:"meta"`

    // --- The Business Data ---
    // Unified Field Map (ISO8583 tags, JSON keys).
    Data map[string]any `cbor:"data"`

    // --- Signaling ---
    Flags uint32 `cbor:"flags"` // System-level signaling (e.g., 0x01 = Sync Probe)

    // --- Fallback & Audit ---
    RawPayload []byte `cbor:"raw"`  // Original wire bytes
    Path       []*Hop `cbor:"path"` // Audit Trail
    TsInit     int64  `cbor:"ts"`   // Timestamp of creation (Unix Nanos)
}
```

```go
type Hop struct {
    GearID uuid.UUID `cbor:"g"`
    PortID uuid.UUID `cbor:"p"`
    TsNano int64     `cbor:"t"`
}
```

#### Field dictionary

| Field | Type | Tag | Description |
| :--- | :--- | :--- | :--- |
| **`FluxID`** | `uuid.UUID` | `id` | Globally unique transactional ID (UUID v7). |
| **`RefFluxID`** | `uuid.UUID` | `ref_id` | Correlation ID (references a Parent signal). |
| **`TraceID`** | `string` | `trace` | W3C Otel Trace ID for distributed observability. |
| **`SrcGearID`** | `uuid.UUID` | `src_id` | The `entity_id` of the originating Gear. |
| **`Metadata`** | `map[string]string` | `meta` | Signal Metadata (Routing flags, protocol headers). |
| **`Data`** | `map[string]any` | `data` | Business fields (ISO8583 tags, user-defined keys). |
| **`Flags`** | `uint32` | `flags` | System-level signaling (e.g., `0x01` = Sync Probe). |
| **`RawPayload`**| `[]byte` | `raw` | The original, bit-perfect wire bytes. |
| **`Path`** | `[]*Hop` | `path` | The hops a message took, for auditability. Recorded, not signed. |
| **`TsInit`** | `int64` | `ts` | Unix Nanoseconds of entry into the Rig. |
 
 ### System signaling (flags)
 
 The **`Flags`** field is a 32-bit bitmask used for high-speed, system-level signaling that must be processed without parsing the business `Data` map.
 
 | Bit | Hex | Name | Description |
 | :--- | :--- | :--- | :--- |
 | `0` | `0x01` | **`FlagSyncProbe`** | **Data-Plane Sync**: Used by the Relentless Handshake to verify data-plane availability. |
 | `1..31` | - | *Reserved* | Reserved for future system orchestration signals. |
 
---


## Signal metadata dictionary

To ensure interoperability across the fleet, **fluxrig** distinguishes between Industry Standard keys and project-specific internal keys.

### W3C trace context (standard)
These keys adhere strictly to the [W3C Trace Context](https://www.w3.org/TR/trace-context/) standard used by OpenTelemetry. They use **no-dot** notation.

| Key | Description | Source |
| :--- | :--- | :--- |
| **`traceparent`** | W3C Trace Parent (`00-{trace_id}-{span_id}-{flags}`). | Ingress / [TCP I/O](gears/io_tcp.md) |
| **`tracestate`** | W3C Trace State (vendor-specific trace data). | Ingress / [TCP I/O](gears/io_tcp.md) |

### fluxrig signal metadata
Internal keys follow the **Dot Notation** convention (`namespace.property`).

| Key | Type | Description | Source |
| :--- | :--- | :--- | :--- |
| **`conn.id`** | Hex String | Unique connection ID (Session) for routing responses. | [TCP I/O](gears/io_tcp.md) |
| **`fluxrig.source`** | String | Name/ID of the gear that emitted the message. | [Architecture](../architecture/gear.md) |
| **`spec.id`** | Hex String | The `entity_id` of the Spec used to parse this message. | [Codec Gear](gears/codec_iso8583.md) |
| **`peer.ip`** | String | IP Address of the remote sender. | [TCP I/O](gears/io_tcp.md) |
| **`peer.port`** | String | Port of the remote sender. | [TCP I/O](gears/io_tcp.md) |
| **`http.method`** | String | HTTP Method (GET, POST) if applicable. | HTTP I/O |
| **`http.path`** | String | URL Path. | HTTP I/O |
| **`iso8583.mti`** | String | Message Type Indicator (e.g., "0200"). | [Codec](gears/codec_iso8583.md) / [IO (Heuristic)](gears/io_iso8583.md) |
| **`iso8583.bitmaps`** | String | Count of bitmaps present (e.g., "1", "2"). | [Codec](gears/codec_iso8583.md) / [IO (Heuristic)](gears/io_iso8583.md) |
| **`iso8583.fields`** | String | List of active fields (e.g., "[2, 3, 4]"). | [Codec](gears/codec_iso8583.md) / [IO (Heuristic)](gears/io_iso8583.md) |
| **`iso8583.raw_header`**| Hex String | Preserved raw framing header (Prefixed with `hex:`). | [ISO8583 I/O](gears/io_iso8583.md) |
| **`iso8583.src_id`** | String | Routing Header Source ID. | [ISO8583 I/O](gears/io_iso8583.md) |
| **`iso8583.dst_id`** | String | Routing Header Destination ID. | [ISO8583 I/O](gears/io_iso8583.md) |
| **`conductor.origin`** | String | Origin stamp for cross-Conductor response routing (`out_response_<origin>`). | [Conductor](gears/conductor.md) |
| **`coatcheck.ttl`** | String | Duration Override (e.g., "5m", "120s"). | [Coat Check](gears/coatcheck.md) |
| **`event.type`** | String | Event ID (e.g., "flux.event.timeout"). | [Coat Check](gears/coatcheck.md) |
| **`timeout.key`** | String | The key that expired. | [Coat Check](gears/coatcheck.md) |
| **`timeout.bucket`** | String | The bucket where the key expired. | [Coat Check](gears/coatcheck.md) |

### Payments glossary (industry standard aliases)

To ensure correlation logic (Coat Check, the Conductor, and the Correlator `[Roadmap]`) functions uniformly regardless of the underlying dialect, we enforce exact standard aliases mapped within the Codec SDL.

| Alias Key | Type | Description |
| :--- | :--- | :--- |
| **`card.pan`** | String | Primary Account Number (Often Field 2). |
| **`card.exp`** | String | Expiration Date (YYMM format). |
| **`routing.bin`** | String | The Bank Identification Number (extrapolated or explicit). |
| **`amount.transaction`** | String | The nominal transaction amount. Kept as a String (with implied decimals, e.g., `000000010000` for 100.00) to strictly avoid floating-point errors. |
| **`trace.stan`** | String | System Trace Audit Number (Often Field 11). |
| **`trace.rrn`** | String | Retrieval Reference Number. |

> [!IMPORTANT]
> **Metadata Integrity Guard**
> To ensure cross-language interoperability and wire compatibility with CBOR (RFC 8949), all `Metadata` values MUST be UTF-8 compliant. 
> - **Binary Data**: Must be hex-encoded and prefixed with `hex:` (e.g., `hex:4e62...`).
> - **Validation**: The system will authoritatively reject messages containing non-UTF-8 metadata at the Bus boundary.

---

## entity_id structure

The **entity_id** identifies a persistent component (Rack, Gear, specific Spec Version). It uses a **128-bit UUID v7** structure that embeds technical hints:
`[ Timestamp (48) ] [ Version (4) ] [ Variant (2) ] [ EntityType (8) ] [ Machine ID Hint (32) ] [ Random (34) ]`

| Prefix (Hex) | Entity Type | Relation to machine_id |
| :--- | :--- | :--- |
| `0x00` | **Reserved** | System Broadcast / Null |
| `0x01` | **Cluster** | Physical infrastructure (HA cluster) |
| `0x02` | **Mixer** | Logical tenant unit |
| `0x03` | **fluxMsg** | Message Instance |
| `0x04` | **Rack** | **Includes machine_id** |
| `0x05` | **Gear** | RackID + Local Index |
| `0x06` | **PortInput** | Sink (Standard In) |
| `0x07` | **PortOutput** | Source (Standard Out) |
| `0x08` | **Wire** | Connection between ports |
| `0x09` | **Snake** | Transport Tunnel |
| `0x0a` | **Scenario** | Config Definition |
| `0x0b` | **Session** | Runtime Context |
| `0x0c` | **SpecVersion**| Immutable Spec Blob ID |

---

## Registry and identity mapping

The Registry manages the `machine_id` allocations that make up the `entity_id`. To support clustering and observability, **machine_ids** are now **128-bit UUIDs**.

### Machine ID policy

| Identity | Persistence | Assignment | Description |
| :--- | :--- | :--- | :--- |
| **uuid.Nil** | **Transient** | Default | Represents a node without an assigned Identity (e.g., during initial handshake or offline mode). |
| **UUID v7** | **Permanent** | Registry | Unique identifier assigned to Mixers and Racks. Includes a 32-bit machine hint for log traceability. |

### Identity assignment strategy

| Component | Strategy | Mechanism | Entity Type |
| :--- | :--- | :--- | :--- |
| **Mixer** | **Persistent** | Configuration or Autonomous | `0x02` |
| **Rack** | **Dynamic** | Registry Assignment | `0x04` |
| **Snake** | **Implicit** | Connection | `0x09` |

**Mixers** use static assignment via configuration to ensure stable leadership.
**Racks** receive their ID dynamically during enrollment. This ID is then embedded into the **entity_id** (Type `0x04`).
**Snakes** (Type `0x09`) represent the **connection path** (NATS Link) from a Rack to a Mixer. They inherit identity from their host.

### Unified registry schema

To provide a single pane of glass for all infrastructure, the Mixer maintains a **Unified Registry** table (`registry`) in its embedded DuckDB. This stores `flux_id` -> `PublicKey` mappings.

**Table: `registry`**

| Column | Type | Description |
| :--- | :--- | :--- |
| `entity_id` | **`UUID`** | **PK**. Full 128-bit entity_id. |
| `type_id` | `USMALLINT` | 2=Mixer, 3=Rack, 8=Snake. |
| `machine_id` | **`UUID`** | The assigned Identity (128-bit). |
| `name` | `TEXT` | Unique Hostname (e.g., `rack-01`). |
| `status` | `TEXT` | `active`, `pending`, `offline`. |
| `version` | `TEXT` | Build Version (e.g., `v0.1.0`). |
| `stats` | `JSON` | Real-time metrics (e.g., CPU, Goroutines). *Note: Transported via CBOR.* |
| `attributes` | `JSON` | Static Metadata. *Note: Transported via CBOR.* |
| `started_at` | `TIMESTAMP` | Process start time. |
| `last_seen` | `TIMESTAMP` | Heartbeat timestamp. |

## fluxMsg vs trace_id

Before diving into distributed tracing concepts elsewhere, it is important to delineate between our two primary identifiers:

*   **`flux_id`**: The physical, unique pointer to the record in *our* internal data store (e.g., DuckDB, parquet). Generated using the **UUID v7** standard for high-speed local issuance with time-ordering.
*   **`trace_id`**: The logical, distributed trace (opentelemetry/w3c) that ties this `flux_id` to external systems traversing outside the fluxrig boundary.