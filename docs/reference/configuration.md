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

## Validation at startup

Both binaries validate their configuration when they load it, and refuse to start on either of these:

* **A duration that does not parse.** Every setting whose value is a duration (its name ends in `_timeout`, `_interval`, `_wait`, `_delay`, `_age`, `_duration`, `_ttl` or `_renew`) is parsed at load time. A typo such as `operation_timeout = "5 seconds"` fails the start and names the field. Before this check, a bad duration could reach a consumer that ignored the parse error and treated zero as real, which fires the timer immediately.
* **A key that maps to no setting.** A misspelled section or field, such as `[observabilitty]`, fails the start and names the key. Before this check, the key was ignored and the process ran with a default the operator meant to override.

Some keys are read by the loader itself rather than by a struct field, and are accepted: the Rack's `rack.bus.*` and legacy `bus.*` aliases, `store.state_file` and `store.scenario_file`, and the Mixer's `mixer.mixer_name`.

Both binaries also refuse to start on these values:

* `snake.ping_interval` that is not a positive duration, or `snake.max_pings_out` below 1.
* `snake.ack_wait` that is not longer than `snake.operation_timeout`.

The Rack also refuses to start on these values:

* `snake.failover_after` or `snake.consumer_replicas` below 0.
* `snake.consumer_missed_heartbeats` below 1.

The Mixer also refuses to start on these values:

* `snake.replicas` below 1, or above 1 without `snake.external`.
* `snake.external` without `snake.url`, or without `registry.lease_enabled`.
* `snake.form_timeout` that is not a positive duration.
* `registry.spec_max_size` below 1 or above 2147483647.
* With `registry.lease_enabled`: `registry.lease_renew` that is more than half of `registry.lease_ttl`, or a `registry.lease_bucket` or `registry.spec_bucket` that is empty or has the name of another registry bucket.
* `observability.storage.type` that is not `local`, `s3`, `nats` or `otlp`. The types `s3`, `nats` and `otlp` also need `registry.lease_enabled`, the settings that their type requires, and a positive `observability.storage.batch_interval` and `observability.storage.commit_timeout`.

## Secret management
**fluxrig** does not store secrets in plaintext TOML. For sensitive values (e.g., database passwords, API keys):

1. Use Environment Variables at runtime (`FLUXRIG_STORE_DATABASE_PASSWORD`).
2. Use the **[State Envelope (Passport)](security_reference.md#identity-envelopes-rackflux)**: secrets are carried in the signed envelope rather than in plaintext TOML. The envelope is cryptographically **signed** (Ed25519); at-rest **encryption** is not yet implemented (see the security reference).

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
| `scenario_file` | `string` | `"scenario.flux"` | The Rack's own copy of the last scenario it applied, in `state_dir` (CBOR, mode `0600`). The Rack resumes it at start without waiting for the Mixer to send it again. Empty keeps no copy. Also settable as `store.scenario_file`. |

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
| `lane_queue_size` | `int` | `1024` | Messages each [hot lane](scenario.md#lanes) wire holds in memory between the gear that emits and the gear that consumes. |
| `lane_send_timeout` | `string` | `"5s"` | How long a gear that emits waits for room in a full hot lane queue before it gets an error. |
| `outbox_max_messages` | `int` | `10000` | Bus-bound emissions parked in memory while the bus is down, forwarded when it returns. When full the oldest is dropped and counted; the gear is never blocked. |
| `outbox_drain_retries` | `int` | `3` | More attempts the drain makes on a parked emission the bus refuses before it moves on. The emission stays parked for the next drain. Only an emission the bus can never accept is dropped. |
| `outbox_order` | `string` | `"subject"` | Drain order. `subject` never forwards an emission ahead of an older one on the same subject. `none` forwards whatever the bus takes, in the order the bus takes it. |
| `outbox_backpressure_timeout` | `string` | `"1s"` | With `subject` order, how long a live emission waits for older parked messages on its subject to drain while the bus is up. Then it goes to the bus, or parks behind them if the wait ends first. The wait slows the gear to the drain's pace, so the outbox does not fill and drop messages. With the bus down, nothing waits. `"0s"` turns the wait off. |
| `cleanup_timeout` | `string` | `"2s"` | Grace given to release resources after draining. |
| `bootstrap_secret` | `string` | `"fluxrig"` | Shared secret this Rack presents on its very first Hello, before it has a Passport of its own. Must match the Mixer's `enrollment.bootstrap_secret`. Once enrolled, the Rack presents its own per-Rack secret instead; this only matters for that first contact. |

#### `[Store]`
Data storage settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `dir` | `string` | `"./data"` | Root directory for data storage. |

#### `[Snake]` (NATS/JetStream)
Configuration for the underlying transport bus.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `url` | `string` | `"nats://localhost:4222"` | NATS Server URL. A comma-separated list names several servers. |
| `domain` | `string` | `"flux"` | **JetStream Domain**. Must match Mixer cluster name. |
| `external` | `bool` | `false` | Set to `true` when `url` names a NATS cluster that the operator runs, as for Mixers with `snake.external`. The Rack then verifies the host of the URL in the server certificate, not `domain`. It also does not delete the ephemeral consumer of a subscription that ended: the consumer expires after `inactive_threshold`. |
| `client_cert_file` | `string` | `""` | Certificate that the Rack presents to a NATS server that asks for a client certificate. Set it together with `client_key_file`, or set neither. |
| `client_key_file` | `string` | `""` | Private key of `client_cert_file`. |
| `credentials_file` | `string` | `""` | NATS credentials file: the NATS user of this Rack, with the permissions that the operator gave it. |
| `ping_interval` | `string` | `"5s"` | Time between the pings that the Rack sends to its NATS server. After `max_pings_out` pings with no answer, the Rack connects to another server of `url`. Must be a positive duration. |
| `max_pings_out` | `int` | `3` | Pings with no answer before the Rack connects to another server. At least 1. |
| `ack_wait` | `string` | `"10s"` | How long a guaranteed-lane or durable consumer waits for an acknowledgement before it delivers the message again. Must be longer than `operation_timeout`, which bounds the processing of one message. |
| `failover_after` | `int` | `2` | JetStream publishes in a row that find no stream, or JetStream requests in a row that a server answers with no JetStream, before the Rack connects to another server of `url`. `0` turns it off. |
| `consumer_replicas` | `int` | `0` | Replica count of the consumer of each guaranteed-lane wire and scenario push. `0` takes the replica count of the stream, read again each time the consumer is created. `1` keeps one replica. |
| `consumer_missed_heartbeats` | `int` | `2` | Consumer heartbeats in a row that a subscription can miss before the Rack replaces its consumer. A heartbeat is due every `operation_timeout`. The new consumer resumes at the first message that the Rack did not settle, so a message can arrive two times. At least 1. |
| `stream_name` | `string` | `"flux-msg"` | Primary JetStream stream name. |
| `connect_timeout` | `string` | `"10s"` | Initial connection timeout. |
| `reconnect_wait` | `string` | `"1s"` | Wait time between reconnect attempts. |
| `convergence_delay` | `string` | `"100ms"` | Safety delay for consumer convergence. |
| `operation_timeout` | `string` | `"5s"` | Deadline for a single bus operation. |
| `subscription_retry_wait` | `string` | `"200ms"` | Wait between attempts to establish a subscription. |
| `subscription_retry_attempts` | `int` | `5` | Attempts before a subscription is reported failed. |
| `initial_retry_wait` | `string` | `"500ms"` | Wait before the first reconnect attempt, with exponential backoff on repeats. |
| `initial_retry_attempts` | `int` | `10` | Attempts before an initial connection is reported failed. |
| `offline_start_timeout` | `string` | `"3s"` | Longest a Rack holding a passport waits for the bus at start before it runs offline. A Rack without a passport keeps the full `initial_retry_*` schedule. `0s` removes the bound. |
| `offline_retry_interval` | `string` | `"5s"` | How often a Rack that started offline probes the bus. When it answers, the Rack restarts its session and goes online. `0s` turns the probe off, and the Rack stays offline until restarted. |
| `store_encryption` | `bool` | `true` | Mixer only. Encrypts the message store of the bus on disk: every message, stream and key-value bucket. See [data at rest](../architecture/security.md#data-at-rest-and-in-logs). |
| `store_cipher` | `string` | `"chacha"` | Mixer only. `chacha` (ChaCha20-Poly1305) or `aes` (AES-GCM). |
| `store_key_file` | `string` | `""` | Mixer only. File holding the store key, at least 32 characters. Empty derives the key from the cluster key. |
| `store_old_key_file` | `string` | `""` | Mixer only. The key the store had before `store_key_file`, for the start that rotates it. |
| `stream_max_age` | `string` | `"24h"` | Mixer only. Oldest message a bus stream keeps. Empty keeps them without an age limit. |
| `stream_max_bytes` | `int` | `1073741824` | Mixer only. Largest size of a bus stream, in bytes. `0` means no limit. |
| `inactive_threshold` | `string` | `"30s"` | Silence after which a consumer is treated as inactive. |
| `root_ca_file` | `string` | `""` | CA bundle verifying the Mixer's bus certificate. Also settable as `rack.bus.root_ca_file`. |
| `insecure_skip_verify` | `bool` | `false` | Skip verification of the bus certificate. Development only. Also settable as `rack.bus.insecure_skip_verify`. |
| `allow_non_tls` | `bool` | `false` | Mixer only. Lets a plaintext client connect alongside a TLS one once `tls_cert_file`/`tls_key_file` are set. Off by default, so configuring TLS actually requires it instead of just offering it. |
| `tls_ca_file` | `string` | `""` | Mixer only. CA that a client certificate is verified against, with `tls_verify`. |
| `tls_verify` | `bool` | `false` | Mixer only. The embedded server requires a client certificate, verified against `tls_ca_file`. Not usable in this release: the router of the Mixer connects to its own server with no client certificate, and the embedded server has no user for each Rack. For client certificates and a user for each Rack, use an external NATS (see [High availability](high_availability.md)). |
| `kv_max_bytes` | `int` | `1073741824` | Mixer only. Largest size of a key-value bucket the Mixer provisions, in bytes. Caps it the same way `stream_max_bytes` caps a stream. |
| `kv_max_value_size` | `int` | `1048576` | Mixer only. Largest single value a key-value bucket accepts, in bytes. |
| `kv_ttl` | `string` | `"0s"` | Mixer only. Expiry for a key-value bucket entry. `0s` means no expiry. |

**Telemetry settings**

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `telemetry.service_name` | `string` | `"flux.rack"` | Service name reported on every span, metric and log. |
| `telemetry.base_subject` | `string` | `"flux.telemetry"` | Bus subject prefix telemetry batches are published on. |
| `telemetry.batch_interval` | `string` | `"5s"` | How often a batch is flushed to the Mixer. On the Mixer it is also how often telemetry is flushed to persistent storage. |
| `telemetry.max_batch_size` | `int` | `512` | Records per batch before it is flushed early. |
| `telemetry.stream_name` | `string` | `"flux-telemetry"` | Name of the JetStream stream that holds the telemetry batches. |
| `telemetry.disabled` | `bool` | `false` | Turns telemetry off. Also settable with `FLUXRIG_DISABLE_TELEMETRY`. |
| `telemetry.metrics.host_enabled` | `bool` | `false` | Collect host metrics (CPU, memory, disk, network). |
| `telemetry.metrics.runtime_enabled` | `bool` | `false` | Collect Go runtime metrics. |
| `telemetry.metrics.bento_enabled` | `bool` | `false` | Collect the metrics of the Bento gear. |

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
| `forward_timeout` | `string` | `"5s"` | With `registry.lease_enabled`, how long a Mixer that does not hold the lease waits for the holder to answer a control write that it forwards. It also bounds the read of the lease for `/health`. Set it to cover a round trip between the sites plus the write of the holder. |
| `follow_interval` | `string` | `"5s"` | With `registry.lease_enabled`, how often a Mixer that does not hold the lease reads the active-scenario pointer. When the pointer changed, the Mixer loads the new scenario. |
| `queue_group` | `string` | `"fluxrig-mixers"` | With `registry.lease_enabled`, the durable queue group that the Mixers use for the Hello and heartbeat subjects, so one Mixer handles each message. The consumers have the name of the group and the subject, and stay when a Mixer stops. Empty makes every Mixer handle every message. Without the lease it has no effect. |

#### Registry-First Identity Model

> [!NOTE]
> **Changed**: the static `machine_id` configuration field is gone from both Rack and Mixer configurations, and is no longer read. **fluxrig** now enforces a **Registry-First Enrollment Model**. 
> 
> *   **Racks**: Dynamically receive a 128-bit `uuid.UUID` Identity during the Enrollment Handshake, which is cryptographically signed and stored in the local `rack.flux` passport.
> *   **Mixers**: Automatically generate a persistent cluster identity (UUID v7) on their first boot, which is recorded in the registry bucket (see `[Registry]`).

#### `[Enrollment]` (Mixer)
Control how new Racks are admitted to the rig.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `auto_adopt` | `bool` | `false` | If `true`, any new Rack connecting will be immediately set to `active`. Use cautiously in production. |
| `push_delay` | `string` | `"1s"` | Delay before pushing the initial scenario to a newly adopted Rack. |
| `bootstrap_secret` | `string` | `"fluxrig"` | Shared secret for zero-config enrollment and identity adoption. It ships with a known value, so a deployment that does not set it is running the documented one. |

#### `[Registry]` (Mixer)
Where the Mixer keeps Rack records, the name index, scenario files and the active-scenario pointer, and how long a Rack counts as online. It also sets the write lease and the shared spec store of several Mixers. The buckets are on the bus: in the store of the embedded NATS server (`snake.store_dir`), or on the external NATS with `snake.external`. See [Identity & registry](registry.md) and [high availability](high_availability.md).

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `bucket` | `string` | `"fluxrig_registry"` | Key-value bucket for records, names, scenario files and the active pointer. |
| `liveness_bucket` | `string` | `"fluxrig_liveness"` | Key-value bucket for presence. Must differ from `bucket`. |
| `replicas` | `int` | `1` | Replica count of the registry, liveness, lease and spec buckets. At least 1. It does not follow `snake.replicas`: set both. With `snake.external`, the Mixer logs a warning when it is lower than `snake.replicas`. |
| `max_value_size` | `int` | `1048576` | Largest record in bytes, headers included. It bounds a scenario file: a larger one is refused on import. |
| `liveness_ttl` | `string` | `"90s"` | How long after its last heartbeat a Rack still counts as online. Three heartbeat intervals by default. |
| `write_retries` | `int` | `3` | How many times a registry write that met a concurrent one is read again and retried before it fails. `0` fails on the first conflict. |
| `sweep_interval` | `string` | `"10m"` | How often the Mixer repairs what a crash between two registry writes left behind. It also sweeps once at startup. |
| `orphan_min_age` | `string` | `"5m"` | How old a registry record must be before the sweep treats it as left behind rather than in progress. |
| `lease_enabled` | `bool` | `false` | Turns on the write lease. One Mixer holds the lease and does every control write: enrollment, approval, removal, status, scenario import and activation, and spec import. The other Mixers forward their control writes to the holder, and every Mixer serves reads. When no Mixer holds the lease, a control write answers 503. With the lease on, every Mixer must use the same cluster key (`store.cluster_key_file`), or it refuses to start. `snake.external` needs it. |
| `lease_bucket` | `string` | `"fluxrig_lease"` | Key-value bucket that holds the lease. Must differ from `bucket` and `liveness_bucket`. |
| `lease_ttl` | `string` | `"30s"` | How long the lease stays valid with no renew. The holder stops its writes when it cannot renew within this time. |
| `lease_renew` | `string` | `"10s"` | How often the holder renews the lease. At most half of `lease_ttl`. |
| `spec_bucket` | `string` | `"fluxrig_specs"` | With `lease_enabled`, the key-value bucket of the spec store that the Mixers share. Must differ from `bucket`, `liveness_bucket` and `lease_bucket`. The first Mixer of the cluster that holds the lease copies its local spec store into the bucket, one time. Then the Mixers do not use their local spec stores. |
| `spec_max_size` | `int` | `1048576` | Largest spec in the shared spec store, in bytes. A larger spec answers 413. Must not be more than the max payload of the NATS server. |

#### `[Ingest]` (Mixer)
Telemetry ingestion buffering and flushing. Neither field is read today: the Mixer flushes telemetry to persistent storage every `telemetry.batch_interval` (default `5s`).

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `flush_interval` | `string` | n/a | **[Roadmap]** Not read today. See `telemetry.batch_interval`. |
| `buffer_size` | `int` | n/a | **[Roadmap]** Not read today. |

#### `[API]`
Mixer REST API settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `port` | `int` | `8090` | **Server Port** for the HTTP/gRPC API listener. |
| `read_header_timeout` | `string` | `"3s"` | Deadline for reading request headers, which bounds a slow-header client. |
| `control_confirm_timeout` | `string` | `"2s"` | How long a simulator control command (`/api/v1/control/sim/{action}`) waits for a gear to acknowledge it before the request answers that nobody is listening. |
| `approval_push_timeout` | `string` | `"10s"` | How long the push of the active scenario to a Rack that an operator approves may take. The push continues if the client that asked for the approval disconnects. |
| `tls_cert_file` | `string` | `""` | Server certificate. The API serves HTTPS when this and `tls_key_file` are both set. |
| `tls_key_file` | `string` | `""` | Server private key. |
| `auth_token` | `string` | `""` | Bearer token required as `Authorization: Bearer <token>` on every route but `/api/v1/health`. The Mixer refuses to start with this unset unless `auth_disabled_dangerously` is set. |
| `auth_disabled_dangerously` | `bool` | `false` | Starts the Mixer with no API authentication. The explicit, named opt-out for a deployment with no `auth_token` configured; logs a warning on every start. |

#### `[Wasm]` (Mixer)
Where signed Wasm gears and the keys that verify them are kept.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `catalog_dir` | `string` | `"./data/wasm"` | Directory holding imported Wasm modules. |
| `trusted_keys_dir` | `string` | `"./data/wasm/keys"` | Public keys a module's signature is checked against. |

#### `[Store]` (Mixer)
Local data storage: telemetry, and the topology rows derived from the active scenario. The registry itself is in `[Registry]`.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `database_file` | `string` | `"flux.duckdb"` | DB filename (relative to store dir). |
| `cluster_key_file` | `string` | `"cluster.key"` | Key filename (relative to store dir). |

#### `[Snake]`
NATS (JetStream) settings: the embedded server, or with `external` the NATS cluster that the operator runs.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `port` | `int` | `4222` | **Server Port** to listen on for NATS connections. Not used with `external`. |
| `url` | `string` | `"nats://localhost:4222"` | URL to advertise to Racks. With `external`, the NATS cluster that the Mixer uses, as a comma-separated list of its servers. Do not embed credentials. `/api/v1/config` removes URL userinfo; use `credentials_file` or client certificates for authentication. |
| `external` | `bool` | `false` | The Mixer starts no embedded server and uses the NATS cluster at `url`. Several Mixers on one cluster give high availability. Needs `registry.lease_enabled`. The Mixer verifies the host of the URL in the server certificate, not `domain`. At start it needs NATS 2.15.0 or later, JetStream for its account and, when `replicas` is more than 1, a cluster: if one of these is missing, the start fails. |
| `replicas` | `int` | `1` | Copies of each stream and bucket that the Mixer provisions. The registry buckets take `registry.replicas`. More than 1 needs `external`. With `external`, the Mixer logs a warning below 3. |
| `form_timeout` | `string` | `"2m"` | With `external`, how long the Mixer tries again while the NATS does not answer, a stream cannot be placed or a bucket cannot be created. Then the start fails. Must be a positive duration. |
| `client_cert_file` | `string` | `""` | Certificate that the Mixer presents to a NATS server that asks for a client certificate. Set it together with `client_key_file`, or set neither. |
| `client_key_file` | `string` | `""` | Private key of `client_cert_file`. |
| `credentials_file` | `string` | `""` | NATS credentials file: the NATS user of this Mixer, with the permissions that the operator gave it. |
| `ping_interval` | `string` | `"5s"` | Time between the pings that each connection of the Mixer sends to its NATS server. After `max_pings_out` pings with no answer, the connection goes to another server. Must be a positive duration. |
| `max_pings_out` | `int` | `3` | Pings with no answer before a connection goes to another server. At least 1. |
| `ack_wait` | `string` | `"10s"` | How long a guaranteed-lane or durable consumer waits for an acknowledgement before it delivers the message again. Must be longer than `operation_timeout` (default `"5s"`). |
| `domain` | `string` | `"flux"` | JetStream Domain Name. |
| `cluster_name` | `string` | `"flux"` | JetStream domain the embedded server reports, and what a client uses to reach a domain-scoped server. |
| `store_dir` | `string` | `"./data/snake"` | Where the embedded server keeps its JetStream store. Empty keeps the default, a `snake` directory under `store.dir`. |
| `stream_name` | `string` | `"flux-msg"` | Name of the primary JetStream stream. |
| `stream_subjects` | `[]string` | `["flux.msg.>", "flux.gear.>", "fluxrig.>"]` | Wildcard subjects to capture. |
| `business_stream_max_age` | `string` | n/a | **[Roadmap]** Not read today. The age limit of the message stream is `stream_max_age`, below. |
| `telemetry_stream_max_age` | `string` | n/a | **[Roadmap]** Not read today. |
| `durable` | `bool` | n/a | **[Roadmap]** Not read today. The bus stores its data on disk. |
| `dlq_enabled` | `bool` | `true` | Route a message no consumer could process to the dead letter stream instead of dropping it. Off keeps the old behavior: log and discard. |
| `dlq_subject` | `string` | `"flux.dlq"` | Subject prefix the dead letter stream captures. A failed message is republished as `<prefix>.<origin-subject>`. |
| `dlq_max_age` | `string` | `"720h"` | How long a dead letter is kept. Empty keeps the stream default (`24h`). |
| `events_max_age` | `string` | `"1h"` | Mixer only. How long the `flux-events` stream keeps a durable event. A reader that was down longer has lost what it missed. |
| `events_max_bytes` | `int` | `268435456` | Mixer only. Largest size of the `flux-events` stream, in bytes. An event takes about 390 bytes, and a connection makes two events. |
| `dlq_publish_timeout` | `string` | `"5s"` | How long one dead letter publish may wait for the server. The Rack reads the same key. A value that is not a duration stops the start. |

> [!NOTE]
> By default the Mixer starts the embedded server itself on every boot. With
> `external = true` it starts no server and uses the NATS cluster at `url`.
> The operator runs that NATS and keeps its store, encryption at rest
> included: `port`, `store_dir`, the `store_*` encryption settings and the
> server TLS settings (`tls_*`, `allow_non_tls`) do not apply, and
> `fluxrig-mixer --repair` refuses to run. See
> [high availability](high_availability.md).

### Dead letter queue

A message no consumer can process is not lost. The Mixer provisions a
`flux-dlq` stream beside the message and telemetry streams, and the bus
republishes the unprocessable message there. It is the answer to the three
paths that used to drop a message with only a log line:

- The payload does not decode.
- The payload decodes but fails validation.
- The handler fails on every redelivery, up to the redelivery bound.

The stored message keeps the original bytes unchanged, so a replay is
byte-identical. The failure context is in headers:

| Header | Meaning |
| :--- | :--- |
| `Flux-DLQ-Origin-Subject` | The subject the message was being consumed from. |
| `Flux-DLQ-Reason` | `decode`, `validate`, or `redelivery-exhausted`. |
| `Flux-DLQ-Deliveries` | The delivery count when it was routed. |
| `Flux-DLQ-At` | When it was routed, in RFC 3339. |

To replay one, read `flux.dlq.>` (with the `nats` tool or a small
subscriber), fix the cause, and republish the original bytes to
`Flux-DLQ-Origin-Subject`. Set `snake.dlq_enabled = false` to keep nothing.

The dead letter stream's subject must not fall under the message stream's
subjects (`flux.msg.>`, `flux.agent.>`, `flux.rack.>`, `flux.ctrl.>`), or
the stream cannot be created. The default `flux.dlq` does not overlap.

### Durable events

The Mixer provisions a `flux-events` stream that captures `flux.evt.>`, for the
[connection events](gears/io_iso8583.md#connection-events) of `io_iso8583`. A
`stream_subjects` value that includes `flux.evt.>` overlaps it, and the Mixer
logs a warning and starts without the stream. Each Rack then keeps its events in
its outbox.

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

The `tier` setting does not select a backend. Where the Mixer keeps the telemetry of the Racks is `observability.storage.type`: see [`[Observability.storage]`](#observabilitystorage).

---

### `[Observability]`
Global observability settings.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | `bool` | `true` | Enable/disable observability collection. |
| `tier` | `string` | `"embedded"` | **[Roadmap]** Not read today. Where the Mixer keeps telemetry is `storage.type`, below. |
| `sampling_rate` | `float` | n/a | **[Roadmap]** Not read today: every trace is kept. |

#### `[Observability.storage]`
Where the Mixer keeps the telemetry of the Racks.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `type` | `string` | `"local"` | `local`, `s3`, `nats` or `otlp`. `local` keeps the telemetry on the disk of each Mixer, as the [embedded tier](#embedded-tier-configuration) describes. With several Mixers, each one then has its own copy, which does not have what arrived while that Mixer was down, and the Mixer logs a warning. `s3`, `nats` and `otlp` need `registry.lease_enabled`. |
| `batch_interval` | `string` | `"15s"` | With `s3`, `nats` or `otlp`, how often the lease holder commits a batch: one file of each kind to `s3` or `nats`, or one export to `otlp`. The other Mixers see the telemetry up to the last batch. |
| `commit_timeout` | `string` | `"30s"` | With `s3`, `nats` or `otlp`, the longest time for one upload or export of a batch. A batch that fails is tried again. |

With `s3`, `nats` and `otlp`, the lease holder is the only Mixer that reads the telemetry stream. It confirms a batch only when the store has it, so a new holder continues from the last confirmed batch. Every Mixer keeps the live metrics of each Rack.

With `s3` and `nats`, a batch that is committed again replaces its own files, so the store does not keep a record two times. Every Mixer answers telemetry queries: it gets the files that a query needs into `store.dir/telemetry`, under the same retention as local files. The holder removes the shared files that are older than the retention.

With `otlp`, the Mixers keep no telemetry, and the telemetry queries of the API answer 501. When a holder stops after the collector got a batch and before the stream got the confirmation, the next holder sends that batch again: that one batch can arrive two times.

Shared telemetry stores contain message-adjacent data. The batches contain Rack logs, metrics and spans from traffic that the Mixers route. Treat each bucket and collector as sensitive.

#### `[Observability.storage.s3]`
A bucket of an S3-compatible object store. The Mixer writes Parquet files to it and creates the bucket when it does not exist.

The Mixer requests SSE-S3 (`AES256`) for each object. The service must support this setting and enforce its encryption policy. Use HTTPS. The Mixer warns when `plaintext` enables HTTP.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `endpoint` | `string` | `""` | Host and port of the service, with no scheme. Required with `type = "s3"`. |
| `bucket` | `string` | `""` | Bucket name. Required with `type = "s3"`. |
| `prefix` | `string` | `"telemetry"` | Put before every object name. |
| `region` | `string` | `""` | Region of the bucket. |
| `access_key_file` | `string` | `""` | File that holds the access key. Empty reads `AWS_ACCESS_KEY_ID`. |
| `secret_key_file` | `string` | `""` | File that holds the secret key. Empty reads `AWS_SECRET_ACCESS_KEY`. |
| `ca_file` | `string` | `""` | CA that verifies the certificate of the service, when a public CA does not. |
| `plaintext` | `bool` | `false` | Uses HTTP instead of HTTPS. Development only. |

#### `[Observability.storage.nats]`
An object store on the NATS of the Mixers. The Mixer writes the same Parquet files as with `s3`. When the Mixers use an external NATS cluster, the operator must configure encryption at rest for its JetStream store.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `bucket` | `string` | `"fluxrig_telemetry"` | Object store bucket. Its index is in the key-value bucket `<bucket>_index`. Both take `snake.replicas` copies. |

#### `[Observability.storage.otlp]`
An OpenTelemetry collector. The Mixer sends each batch as OTLP over HTTP, with JSON bodies. Use an `https://` endpoint. An `http://` endpoint sends telemetry without encryption, and the Mixer warns at startup.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `endpoint` | `string` | `""` | Base URL of the collector, for example `https://collector.example:4318`. The Mixer adds the paths `/v1/logs`, `/v1/metrics` and `/v1/traces`. Required with `type = "otlp"`. |
| `ca_file` | `string` | `""` | CA that verifies the certificate of the collector, when a public CA does not. |
| `headers_file` | `string` | `""` | File with extra request headers, one `Name: value` per line, for example an authorization token. |

---

### Embedded tier configuration

#### `[Observability.embedded]`
Settings for the Embedded tier (DuckDB + Parquet).

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `data_dir` | `string` | n/a | **[Roadmap]** Not read today. Telemetry is stored under the directory of `[Store]`. |
| `flush_interval` | `string` | n/a | **[Roadmap]** Not read today. See `telemetry.batch_interval`. |
| `retention_days` | `int` | `30` | Delete Parquet files older than N days. Read by the retention sweep. When it is positive it wins over `rotation.max_age_days`. |

The retention sweep runs once when the Mixer starts and then at `rotation.check_interval`. It deletes `.parquet` files only, by the timestamp in the file name, falling back to the file's modification time. It does not delete rows from the DuckDB `telemetry_*` tables, so those tables keep their rows and must be managed separately.

#### `[Observability.embedded.storage]` **[Roadmap]**
Storage format and organization. None of these settings is read today: telemetry is written as Parquet.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `format` | `string` | `"parquet"` | Storage format: `parquet`, `json`. |
| `partition_by` | `string` | `"hour"` | Time partitioning: `hour`, `day`. |
| `compression` | `string` | `"zstd"` | Compression: `zstd`, `snappy`, `none`. |

#### `[Observability.embedded.rotation]`
Retention settings for local Parquet files. `check_interval` and `max_age_days` are read today. `policy` and `max_size_gb` are **[Roadmap]**.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `policy` | `string` | `"time"` | **[Roadmap]** Rotation policy: `time`, `size`, `both`. Retention is age-based today. |
| `max_age_days` | `int` | `30` | Delete data older than N days. Used only when `retention_days` is not positive. |
| `max_size_gb` | `int` | `10` | **[Roadmap]** Not read today. A size cap is not implemented. |
| `check_interval` | `string` | `"1h"` | How often the retention sweep runs. The sweep also runs once at Mixer startup. |

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

# Where telemetry is stored is set by [store], and how often it is flushed by
# telemetry.batch_interval.
```

---

### OTLP export

The Mixer exports telemetry to an OpenTelemetry collector with `observability.storage.type = "otlp"`, set in [`[Observability.storage.otlp]`](#observabilitystorageotlp). The export is OTLP over HTTP, with JSON bodies. OTLP over gRPC is not built.

There is no `[Observability.otlp]` section. A file that sets it fails the start with an unknown key.

#### OTLP example

```toml
[registry]
lease_enabled = true

[observability.storage]
type = "otlp"

[observability.storage.otlp]
endpoint = "https://collector.example:4318"
headers_file = "/etc/fluxrig/otlp-headers"
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

> [!NOTE]
> **Every route here requires authentication.** Send `Authorization: Bearer <api.auth_token>`. The Mixer refuses to start with no token configured, unless `api.auth_disabled_dangerously` is set explicitly (not recommended). The one exception is `/api/v1/health`, which stays open with no token.

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
curl -H "Authorization: Bearer $API_TOKEN" \
  "http://mixer:8090/api/v1/telemetry/logs?level=error&since=2025-12-21T00:00:00Z&limit=50"
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
