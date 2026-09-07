---
slug: /reference/platform/configuration
title: Platform configuration
---

# Platform configuration

This section details the configuration files used by the **fluxrig** platform.

## Configuration precedence & environment variables

To support modern containerized deployments (Docker/Kubernetes), **fluxrig** merges configuration from multiple sources, evaluated in the following order of precedence (highest to lowest):

1.  **CLI Flags** (`--level debug`)
2.  **Environment Variables** (`FLUXRIG_LEVEL=debug`)
3.  **Config File** (`fluxrig.toml`)
4.  **Static Defaults**

The environment variable mapping replaces dots (`.`) with underscores (`_`) and uppercases the key. For example, `base.name` becomes `FLUXRIG_BASE_NAME`.

Top-level overrides for troubleshooting:
* `FLUXRIG_TRACE=true`: Forces all logging to TRACE level (high volume).
* `FLUXRIG_DEBUG=true`: Forces all logging to DEBUG level.
* `FLUXRIG_DISABLE_TELEMETRY=true`: Master switch to stop all self-monitoring reporting.

## Secret management
**fluxrig** does not store secrets in plaintext TOML. For sensitive values (e.g., database passwords, API keys):

1. Use Environment Variables at runtime (`FLUXRIG_STORE_DATABASE_PASSWORD`).
2. Use the **[State Envelope (Passport)](security_reference.md#identity-envelopes-stateflux)**: secrets are carried in the signed envelope rather than in plaintext TOML. The envelope is cryptographically **signed** (Ed25519); at-rest **encryption** is not yet implemented (see the security reference).

---

## Rack configuration (`fluxrig`)

The static binary `fluxrig` loads its configuration from `fluxrig.toml` by default.

### File: `fluxrig.toml`

Configuration is namespaced to separate global/shared settings from component-specific ones.

#### `[Logging]` (Rack)
Shared logging content.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `level` | `string` | `"info"` | Verbosity: `debug`, `info`, `warn`, `error`. |
| `filename` | `string` | `"logs/fluxrig.log"` | Path to log file. |
| `max_size_mb` | `int` | `100` | Max size in MB before rotation. |
| `max_backups` | `int` | `7` | Max number of old log files to keep. |
| `compress` | `bool` | `true` | Compress rotated log files (gzip). |
| `trace` | `bool` | `false` | Hard-override to enable trace level. |
| `debug` | `bool` | `false` | Hard-override to enable debug level. |

**Throttling settings**

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `throttling.enabled` | `bool` | `true` | Toggle log throttling protection. |
| `throttling.rate` | `float`| `500.0` | Max lines per second per gear. |
| `throttling.burst` | `int` | `50` | Temporary log burst capacity. |

#### `[Base]` (Common)
Foundational identity and path settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `name` | `string` | `"node-01"` | **Unique Name**. Claims this specific identity. |
| `state_dir` | `string` | `"./data"` | Root directory for state and WAL. |
| `state_file` | `string` | `"rack.flux"` | **Signed State Bundle** (CBOR). Persists Identity, Config, and Secrets. |

#### `[Rack]`
Rack-specific settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `name_prefix` | `string` | `"node-"` | **Ephemeral Prefix**. Used if `base.name` is empty. |
| `ip` | `string` | `""` | **Manual IP Override**. If empty, the Rack autodetects its local primary IP via `netutil`. |
| `max_hops` | `int` | `64` | **Max Routing Hops**. Prevents infinite loops. |
| `max_payload_size`| `int` | `2097152` | **Max Payload Size (Bytes)**. Default: 2MB. |
| `enrollment_timeout` | `string` | `"15s"` | Timeout for enrollment handshake. |
| `heartbeat_interval` | `string` | `"30s"` | Frequency of Registry heartbeats. |
| `enrollment_interval` | `string` | `"2s"` | Wait between enrollment attempts while a Rack is not yet adopted. |
| `handshake_interval` | `string` | `"500ms"` | Re-emission of the connectivity probes sent before gears are activated. |
| `convergence_timeout` | `string` | `"5s"` | How long a Rack waits for every wire to confirm reachable before giving up on the convergence phase. |
| `drain_timeout` | `string` | `"35s"` | How long in-flight messages are given to finish on shutdown before gears are stopped. |
| `cleanup_timeout` | `string` | `"2s"` | Grace given to release resources after draining. |

#### `[Store]`
Data storage settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `dir` | `string` | `"./data"` | Root directory for data storage. |

#### `[Snake]` (NATS/JetStream)
Configuration for the underlying transport bus.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `url` | `string` | `"nats://localhost:4222"` | NATS Server URL. |
| `domain` | `string` | `"flux"` | **JetStream Domain**. Must match Mixer cluster name. |
| `stream_name` | `string` | `"flux-msg"` | Primary JetStream stream name. |
| `connect_timeout` | `string` | `"10s"` | Initial connection timeout. |
| `reconnect_wait` | `string` | `"1s"` | Wait time between reconnect attempts. |
| `convergence_delay` | `string` | `"100ms"` | Safety delay for consumer convergence. |
| `operation_timeout` | `string` | `"5s"` | Deadline for a single bus operation. |
| `subscription_retry_wait` | `string` | `"200ms"` | Wait between attempts to establish a subscription. |
| `subscription_retry_attempts` | `int` | `5` | Attempts before a subscription is reported failed. |
| `inactive_threshold` | `string` | `"30s"` | Silence after which a consumer is treated as inactive. |
| `root_ca_file` | `string` | `""` | CA bundle verifying the Mixer's bus certificate. Also settable as `rack.bus.root_ca_file`. |
| `insecure_skip_verify` | `bool` | `false` | Skip verification of the bus certificate. Development only. Also settable as `rack.bus.insecure_skip_verify`. |

**Telemetry settings**

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `telemetry.service_name` | `string` | `"flux.rack"` | Service name reported on every span, metric and log. |
| `telemetry.base_subject` | `string` | `"flux.telemetry"` | Bus subject prefix telemetry batches are published on. |
| `telemetry.batch_interval` | `string` | `"5s"` | How often a batch is flushed to the Mixer. |
| `telemetry.max_batch_size` | `int` | `512` | Records per batch before it is flushed early. |

#### Example

```toml
[base]
name = "rack-nyc-01"
state_dir = "/var/lib/fluxrig/data"

[logging]
level = "debug"

[snake]
url = "nats://nats.fluxrig.internal:4222"
```

**Note on Log Files**:

*   Rack writes to `<log_dir>/fluxrig-rack.log`.
*   CLI writes to `<log_dir>/fluxrig-cli.log` (if enabled).

---

## Mixer configuration (`fluxrig-mixer`)

The dynamic binary `fluxrig-mixer` loads its configuration from `fluxrig-mixer.toml`.

### File: `fluxrig-mixer.toml`
Configuration is namespaced into granular sections.

#### `[Logging]` (Mixer)
Shared logging content.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `level` | `string` | `"info"` | Verbosity: `debug`, `info`, `warn`, `error`. |
| `filename` | `string` | `"logs/mixer.log"` | Path to log file. |
| `max_backups` | `int` | `7` | Max number of old log files to keep. |

#### `[Mixer]`
General Mixer settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `max_hops` | `int` | `64` | **Global Max Hops**. |
| `max_payload_size`| `int` | `2097152` | **Global Max Payload (Bytes)**. |
| `startup_scenario` | `string` | `""` | Scenario reference to load on startup. |
| `scenario_wait_timeout` | `string` | `"15s"` | Time to wait for Racks to converge when activating a scenario. |

#### Registry-First Identity Model

> [!NOTE]
> **Changed**: the static `machine_id` configuration field is gone from both Rack and Mixer configurations, and is no longer read. **fluxrig** now enforces a **Registry-First Enrollment Model**. 
> 
> *   **Racks**: Dynamically receive a 128-bit `uuid.UUID` Identity during the Enrollment Handshake, which is cryptographically signed and stored in the local `state.flux` passport.
> *   **Mixers**: Automatically generate a persistent cluster identity (UUID v7) on their first boot, which is maintained in the internal Unified Registry (`registry`).

#### `[Enrollment]` (Mixer)
Control how new Racks are admitted to the rig.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `auto_adopt` | `bool` | `false` | If `true`, any new Rack connecting will be immediately set to `active`. Use cautiously in production. |
| `push_delay` | `string` | `"1s"` | Delay before pushing the initial scenario to a newly adopted Rack. |
| `bootstrap_secret` | `string` | `"fluxrig"` | Shared secret for zero-config enrollment and identity adoption. It ships with a known value, so a deployment that does not set it is running the documented one. |

#### `[Ingest]` (Mixer)
Control telemetry ingestion buffering and flushing.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `flush_interval` | `string` | `"5s"` | Interval for flushing telemetry to persistent storage. |
| `buffer_size` | `int` | `1024` | Internal channel buffer size for telemetry ingestion. |

#### `[API]`
Mixer REST API settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `port` | `int` | `8090` | **Server Port** for the HTTP/gRPC API listener. |
| `read_header_timeout` | `string` | `"3s"` | Deadline for reading request headers, which bounds a slow-header client. |

#### `[Wasm]` (Mixer)
Where signed Wasm gears and the keys that verify them are kept.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `catalog_dir` | `string` | `"./data/wasm"` | Directory holding imported Wasm modules. |
| `trusted_keys_dir` | `string` | `"./data/wasm/keys"` | Public keys a module's signature is checked against. |

#### `[Store]` (Mixer)
Data storage settings (Analytics/Registry).

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `database_file` | `string` | `"fluxrig.duckdb"` | DB filename (relative to store dir). |
| `cluster_key_file` | `string` | `"cluster.key"` | Key filename (relative to store dir). |

#### `[Snake]`
Embedded NATS Server (JetStream) settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `port` | `int` | `4222` | **Server Port** to listen on for NATS connections. |
| `url` | `string` | `"nats://localhost:4222"` | URL to advertise to Racks. |
| `domain` | `string` | `"flux"` | JetStream Domain Name. |
| `stream_name` | `string` | `"flux-msg"` | Name of the primary JetStream stream. |
| `stream_subjects` | `[]string` | `["flux.msg.>", "flux.gear.>", "fluxrig.>"]` | Wildcard subjects to capture. |
| `business_stream_max_age` | `string` | `"720h"` | Data retention time for business logic streams. |
| `telemetry_stream_max_age` | `string` | `"24h"` | Data retention time for telemetry streams. |
| `durable` | `bool` | `false` | Enable disk-durable JetStream persistence. |

> [!NOTE]
> The Mixer starts this server itself on every boot. There is no setting that
> points it at an existing NATS deployment, so these fields move the bus, they
> do not replace it.

#### Example (Mixer)

```toml
[base]
name = "mixer-prod-01"

[logging]
level = "info"

[mixer]
startup_scenario = "scenario_prod.yaml"

[api]
port = 8090

[store]
dir = "./data"

[snake]
port = 4222
url = "nats://localhost:4222"
domain = "flux"
```

---

### [Roadmap] Observability tiers

The platform is designed to support multiple observability tiers for various scales. In **the current release**, only the **Embedded Tier** is fully implemented.

| Tier | Backend(s) | Use Case | Status |
|------|------------|----------|--------|
| `embedded` | DuckDB + Parquet | Zero deps, development, edge | **Stable** |
| `standard` | Arc + OpenSearch | Team deployments | **[Roadmap]** |
| `enterprise` | ClickHouse + SigNoz | Petabyte scale | **[Roadmap]** |

---

### `[Observability]`
Global observability settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | `bool` | `true` | Enable/disable observability collection. |
| `tier` | `string` | `"embedded"` | Backend tier: `embedded` (standard), `otlp` (collector). |
| `sampling_rate` | `float` | `1.0` | Trace sampling rate (0.0 - 1.0). |

---

### Embedded tier configuration

#### `[Observability.embedded]`
Settings for the Embedded tier (DuckDB + Parquet).

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `data_dir` | `string` | `"./data/telemetry"` | Directory for telemetry storage. |
| `flush_interval` | `string` | `"5s"` | Interval to flush data to disk. |
| `retention_days` | `int` | `30` | Days of telemetry kept in DuckDB before it is aged out. |

#### `[Observability.embedded.storage]`
Storage format and organization.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `format` | `string` | `"parquet"` | Storage format: `parquet`, `json`. |
| `partition_by` | `string` | `"hour"` | Time partitioning: `hour`, `day`. |
| `compression` | `string` | `"zstd"` | Compression: `zstd`, `snappy`, `none`. |

#### `[Observability.embedded.rotation]`
Rotation and retention policies.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `policy` | `string` | `"time"` | Rotation policy: `time`, `size`, `both`. |
| `max_age_days` | `int` | `30` | Delete data older than N days. |
| `max_size_gb` | `int` | `10` | Max total storage size in GB. |
| `check_interval` | `string` | `"1h"` | How often to check rotation rules. |

#### `[Store]` (embedded)

- `dir`: Root directory for data storage (e.g., `./data`).
- `wal_max_size_mb`: Max size for Write-Ahead Log.
- `database_file`: Filename for DuckDB file.

The **flux_id** (serialized as `flux_id` in JSON and database schemas) is the unique 128-bit identifier used to trace a specific business signal across its entire lifecycle, from edge ingestion to cloud archival.

#### Embedded storage schema

Data is organized into **telemetry** (system signals) and **messages** (business data):

```
./data/
├── telemetry/               # OTel signals (system observability)
│   ├── traces/
│   │   └── 2025/12/21/14/traces_001.parquet
│   ├── logs/
│   │   └── 2025/12/21/14/logs_001.parquet
│   └── metrics/
│       └── 2025/12/21/14/metrics_001.parquet
└── messages/                # Business data (fluxMsg)
    ├── generic/             # Default schema (raw_msg only)
    │   └── 2025/12/21/14/fluxmsg_001.parquet
    └── fluxSpec/            # Spec-driven schemas
        └── visa-v1/         # Spec-specific subfolder
            └── 2025/12/21/14/visa_001.parquet
```

**Parquet Schema (Traces)**:

| Column | Type | Description |
|--------|------|-------------|
| `ts` | `TIMESTAMP` | Event timestamp |
| `trace_id` | `VARCHAR` | W3C TraceContext ID |
| `span_id` | `VARCHAR` | Span ID |
| `parent_span_id` | `VARCHAR` | Parent Span ID |
| `flux_id` | `UUID` | UUID v7 business ID |
| `entity_id` | `UUID` | Unified EntityID (128-bit UUID v7) |
| `entity_name` | `TEXT` | Human-readable Hostname (e.g., `rack-sfo-01`) |
| `span_name` | `VARCHAR` | Operation name |
| `duration_ms` | `FLOAT` | Duration in milliseconds |
| `status` | `ENUM` | `ok`, `error` |
| `attributes` | `JSON` | Additional attributes |

**Parquet Schema (Logs)**:

| Column | Type | Description |
|--------|------|-------------|
| `ts` | `TIMESTAMP` | Log timestamp |
| `entity_id` | `UUID` | Unified EntityID |
| `entity_name` | `TEXT` | Human-readable Hostname |
| `flux_id` | `UUID` | Business flow ID |
| `level` | `ENUM` | `debug`, `info`, `warn`, `error` |
| `message` | `VARCHAR` | Log message |
| `attributes` | `JSON` | Structured fields |

**Parquet Schema (Metrics)**:

| Column | Type | Description |
|--------|------|-------------|
| `ts` | `TIMESTAMP` | Metric timestamp |
| `name` | `VARCHAR` | Metric name (e.g., `gear_latency_ms`) |
| `entity_id` | `UUID` | Unified EntityID |
| `entity_name` | `TEXT` | Human-readable Hostname |
| `type` | `ENUM` | `counter`, `gauge`, `histogram` |
| `value` | `DOUBLE` | Metric value |
| `labels` | `JSON` | Dimension labels |

**Parquet Schema (fluxMsg / Data)**:

| Column | Type | Description |
|--------|------|-------------|
| `ts` | `TIMESTAMP` | Message timestamp |
| `trace_id` | `VARCHAR` | Associated trace |
| `flux_id` | `UUID` | Primary business ID |
| `entity_id` | `UUID` | Node EntityID |
| `entity_name` | `TEXT` | Hostname |
| `src_gear_id` | `VARCHAR` | Source Gear |
| `dst_gear_id` | `VARCHAR` | Destination Gear |
| `msg_type` | `VARCHAR` | Message type identifier |
| `direction` | `ENUM` | `inbound`, `outbound`, `internal` |
| `status` | `ENUM` | `ok`, `error` |
| `raw_msg` | `BLOB` | Full CBOR payload |
| `promoted` | `JSON` | Promoted fields (spec-driven) |

#### Embedded example

```toml
[observability]
enabled = true
tier = "embedded"
sampling_rate = 1.0

[observability.embedded]
# Data_dir is managed by [store] config
flush_interval = "5s"

[observability.embedded.storage]
format = "parquet"
partition_by = "hour"
compression = "zstd"

[observability.embedded.rotation]
policy = "both"
max_age_days = 30
max_size_gb = 10
check_interval = "1h"
```

---

### OTLP tier configuration (vendor-neutral)

For integration with any OpenTelemetry-compatible backend (Datadog, Jaeger, Grafana Tempo, etc.).

#### `[Observability.otlp]`
OpenTelemetry Protocol exporter settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `endpoint` | `string` | `""` | OTLP gRPC endpoint (e.g., `127.0.0.1:4317`). |
| `endpoint_http` | `string` | `""` | OTLP HTTP endpoint (alternative to gRPC). |
| `headers` | `map` | `{}` | Custom headers (e.g., API keys). |
| `insecure` | `bool` | `false` | Skip TLS verification. |
| `timeout` | `string` | `"30s"` | Request timeout. |

#### OTLP example

```toml
[observability]
enabled = true
tier = "otlp"

[observability.otlp]
endpoint = "otel-collector:4317"
insecure = false
headers = { "api-key" = "${OTEL_API_KEY}" }

# Optional: also store raw fluxMsg locally
[observability.otlp.messages]
enabled = true
data_dir = "./data/messages"
```

---

## CLI query commands

The `fluxrig` CLI provides commands for querying telemetry data.

### `fluxrig logs`
Query log entries.

```bash
# Filter by severity
fluxrig logs --min-level error --since 1h

# Filter by entity
fluxrig logs --entity rack-nyc-01 --limit 100
```

| Flag | Type | Description |
|------|------|-------------|
| `--min-level` | `string` | Minimum log level: `debug`, `info`, `warn`, `error`. |
| `--since` | `duration` | Time window (e.g., `1h`, `24h`) or ISO timestamp. |
| `--until` | `string` | End time (ISO timestamp). |
| `--entity` | `string` | Filter by Entity Name. |
| `--limit` | `int` | Max results (default: 50). |
| `--api-url` | `string` | Mixer API URL (default: `http://localhost:8090`). |

### `fluxrig metrics`
Query metrics data.

```bash
# Filter by metric name
fluxrig metrics --name flux.rack.heartbeats_sent

# Filter by entity
fluxrig metrics --entity mixer-01
```

| Flag | Type | Description |
|------|------|-------------|
| `--name` | `string` | Filter by metric name. |
| `--entity` | `string` | Filter by Entity Name. |
| `--since` | `duration` | Start time. |
| `--until` | `string` | End time. |
| `--limit` | `int` | Max results (default: 50). |
| `--api-url` | `string` | Mixer API URL (default: `http://localhost:8090`). |


---

## REST API endpoints

The Mixer exposes telemetry query endpoints via a REST API.

> [!WARNING]
> **API Security Limitation (Current Release)**: The current release of the Control Plane API does NOT implement authentication or authorization. It is strictly intended for use within trusted, isolated management networks. Exposing this port to the public internet will allow unauthorized scenario activation and data exfiltration.

### Logs API

```
GET /api/v1/telemetry/logs
```

**Query Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `level` | `string` | Filter by level. |
| `since` | `RFC3339` | Start time. |
| `until` | `RFC3339` | End time. |
| `rack_id` | `string` | Filter by Rack. |
| `gear_id` | `string` | Filter by Gear. |
| `q` | `string` | Text search. |
| `limit` | `int` | Max results. |
| `offset` | `int` | Pagination offset. |

**Example**:
```bash
curl "http://mixer:8090/api/v1/telemetry/logs?level=error&since=2025-12-21T00:00:00Z&limit=50"
```

### Traces API `[Roadmap]`

Spans are collected and written to DuckDB and Parquet, and are queryable there
today. The Mixer does not serve them over the API: `GET /api/v1/telemetry/{type}`
answers `logs` and `metrics` and nothing else. Until it does, read them from the
store, which the [telemetry reference](telemetry_analytics.md) shows how to do.

### Metrics API

```
GET /api/v1/telemetry/metrics
```

**Query Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `since` | `RFC3339` | Start time. |
| `until` | `RFC3339` | End time. |
| `group_by` | `string` | Aggregation key. |
| `interval` | `string` | Time bucket: `1m`, `5m`, `1h`. |