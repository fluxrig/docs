---
title: Security architecture
slug: /architecture/security
---

# Security architecture

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

**fluxrig** targets the most hostile network environments (Zero Trust) and the most sensitive data workloads (for example, PCI-DSS, HIPAA). The security model relies on **Network Isolation** and **Sovereign Identity**. It never compromises the integrity of the transactional processing path.

## Network isolation (inbound zero)

To protect infrastructure from external threats, we implement a strict **"Inbound Zero"** policy for the management plane.

### Invisible infrastructure
By moving all orchestration and management logic to an outbound-only tunnel, **fluxrig** removes standard vectors (SSH, HTTP, SNMP) as public entry points. This sharply reduces the network attack surface. External scans effectively cannot see the Rack.

*   **Outbound connection**: All control messages (registry updates, orchestration) and telemetry travel through a persistent, outbound-only connection to the bus: the NATS server embedded in the Mixer, or, with `snake.external = true`, the [NATS](https://docs.nats.io) cluster that the operator runs. It runs over TLS when the server has a certificate.
*   **Data Plane Isolation**: The Rack opens only listening ports explicitly defined by its I/O Gears (for example, a specific TCP socket for protocol ingestion). The Rack isolates these ports strictly from its internal administration and telemetry bus.

### The identity registry
The **Identity Registry** is the foundation of the Unified Control Plane. It maintains the definitive mapping of all Racks, Gears, and Wires in the cluster. Every message therefore carries a cryptographic link to a verified entity.

### Rack identity
A Rack proves who it is with its enrollment, not with its connection:

- **Enrollment**: A new Rack presents the bootstrap secret (`rack.bootstrap_secret`, which must match the Mixer's). After approval, the Mixer issues a secret for that Rack alone, and the Rack sends it in each heartbeat.
- **Passport**: The Mixer signs the identity of the Rack with its Ed25519 cluster key, in `rack.flux`. The Rack keeps the key of the first passport it accepts, and refuses a passport that another key signed.
- **One connection**: The connection carries many independent streams of messages.

What the connection proves depends on the mode of the bus:

- **Embedded mode** (the default): the connection does not authenticate the Rack. The embedded server checks no client certificate and has no users, so a client that reaches its port can read and write every subject, and use the JetStream API.
- **External mode** (`snake.external = true`): the NATS cluster of the operator can authenticate each Rack, by a client certificate or a NATS credentials file, and limit it to the subjects it uses. See [the bus in transit](#the-bus-in-transit).

---

## Data at rest and in logs

What the system writes to disk, and what protects it.

| Where | What is there | Protection |
| :--- | :--- | :--- |
| A Rack's memory | Messages on [hot wires](../reference/scenario.md#lanes), and the `memory` ticket store of the Conductor | Nothing is written to a disk. Hardening the process (memory locked against swap, core dumps disabled) is the operator's |
| The Mixer's message store (`snake/jetstream` in its data directory) | Messages on guaranteed wires, which includes every message that travels between Racks, the enrollment and control traffic of the bus, and the key-value buckets | Encrypted on disk by default |
| Logs, on the Rack and shipped to the Mixer | Log lines | A message payload is never logged at INFO or DEBUG |

### Encrypting the message store

`snake.store_encryption` is on by default. The Mixer derives the key from its cluster key, so nothing has to be configured. The store uses ChaCha20-Poly1305 for encryption, or AES-GCM if `snake.store_cipher` says `aes`.

What it protects: the files on the disk, a copy of the store directory, and a backup taken without the key. What it does not protect: the Mixer holds the key and decrypts on read. A client of the bus sees clear data. Someone with the running Mixer process sees clear data. The derived key lives beside the data (`cluster.key` is in the same data directory), so it does not protect against theft of the whole disk. For production, give the Mixer a key from elsewhere:

*   **An operator key.** `snake.store_key_file` names a file with the key, at least 32 characters. Keep it outside the Mixer's data directory with mode `0600`. The Mixer does not start with a key file it cannot read or that is too short.
*   **Turning it on for an existing store.** The first start with a key converts the store, with no message lost. A copy or a backup taken before that start stays as it was.
*   **A store that was encrypted does not open without its key**. The Mixer refuses to start, with an error that says so, rather than open an empty store. A wrong key fails the start in the same way.
*   **Changing the key.** Save the key the store uses now, for example `fluxrig keys store-key /path/to/cluster.key > store-old.key` for a derived one. Set `snake.store_old_key_file` to that file next to the new key, and start the Mixer: it re-encrypts the store. Later starts do not need the old key file.

These settings apply to the embedded server. In the external mode the Mixer keeps no message store: the NATS cluster holds the messages and the buckets, and its store, encryption at rest included, belongs to the operator.

The retention of a bus stream observes bounds by `snake.stream_max_age` and `snake.stream_max_bytes`.

The setting covers the message store of the bus. It does not encrypt the Mixer analytics database (DuckDB) or its Parquet files. The logs, spans and metrics of the Mixer and the Racks end up there.

Those files underwent a search with a real ISO 8583 authorization: the `19_card_data_iso` test sends one with the card number in DE 2 through two Racks. It reads the database and every Parquet file through DuckDB. It finds the card number in none of them, at the info, debug and trace log levels. The log of the codec names the fields of a message and not their values. The spans of a message carry its identifier and its subject. The search covers the card number of that flow only. It did not search track data, the PIN block or the card verification value.

### The bus in transit

What the bus checks depends on its mode. `snake.root_ca_file` verifies the server certificate in both modes.

**Embedded mode (the default).** The Mixer serves the bus over TLS (`snake.tls_cert_file`, `snake.tls_key_file`). With a certificate it refuses a connection without TLS, unless `snake.allow_non_tls` is set. This is server TLS only:

- The embedded server has no users and no permissions. Anyone who can reach the port of the bus can connect and subscribe to every subject.
- It does not authenticate a client by its certificate. The Mixer connects to its own server with no client certificate, so `snake.tls_verify` is not usable in this mode.

Keep the port on a network that only your Racks and your Mixer can reach.

**External mode (`snake.external = true`).** The Mixers and the Racks connect to a NATS cluster that the operator runs. That cluster decides which clients connect and what each client can do. A Mixer or a Rack proves who it is in one of two ways:

- **A client certificate.** The client presents `snake.client_cert_file` and `snake.client_key_file` to a NATS server that asks for a client certificate. Set both or neither. With `verify_and_map` in the TLS block of the NATS server, NATS maps the subject of the certificate to a user of its authorization block.
- **A credentials file.** `snake.credentials_file` is a NATS credentials file, with the user and the permissions that the operator gave that client.

#### Permissions for each Rack

`fluxrig scenario permissions` writes one NATS user for each Rack of one or more scenario files. Each user can use only the subjects that its Rack uses: its own messages, events and notifications, the wires it consumes, the events its gears read and the buckets they use. Put the users in the authorization block of the external NATS. Write them again when a scenario changes. The subjects and the command are in the [security reference](../reference/security_reference.md#nats-users-for-each-rack).

Forwarded control writes use `flux.mixer.control.>` and carry no caller identity. Keep this subject outside Rack permissions. See [the security reference](../reference/security_reference.md#nats-users-for-each-rack).

The replies and the notifications of a Rack go on subjects that carry its name:

- Each Rack has its own reply inbox, `_INBOX_<rack>`. The Mixer keeps `_INBOX`.
- The Mixer answers an enrollment on `flux.agent.enrollment.<rack>.<nonce>`.
- The Mixer sends a Rack its status, its commands and its passport on `flux.agent.notify.<rack>.<machine-id>`.

Thus the user of a Rack cannot read the replies or the passport of another Rack. NATS refuses a publish or a subscribe that a user is not allowed, and the client logs the refusal at ERROR, with the connection name.

The permissions of a Rack do not refuse these three things:

- **Telemetry under the name of another Rack.** A Rack can publish on every telemetry subject. The Mixer attributes each log by its payload, not by its subject.
- **A message on the subject of a wire that the Rack consumes.** The Rack publishes a sync probe there before its gears start. The other consumers of that wire skip a probe, but they take any other message.
- **The description of a consumer of another Rack, on a stream that both Racks use, when the Rack knows the name of the consumer.** The description gives the filter subject and the position of the consumer, not its messages. A Rack looks up its own consumers when it misses heartbeats, and the names of the consumers are random.

### Logs

A gear never writes the payload of a message to a log at INFO or DEBUG. At TRACE it writes the payload, with card numbers written as digits masked to their first six and last four. Use TRACE only for development: the code does not recognize a number in a binary encoding, such as packed BCD, and does not mask it. Never turn TRACE on with real cardholder data.

---

## Sovereign identity (the passport)

A Rack identity does not depend on a live connection to the Mixer: **fluxrig** uses a **Sovereign Identity** model.

*   **The Passport (`rack.flux`)**: The Rack does not require a real-time connection to the Mixer to check its own integrity. It holds a signed state bundle (The Passport) on-site.
*   **Cluster Authority**: The root of trust is the **Cluster Authority Key**. In the current release, this is a **file-based Ed25519 keypair**.
*   **Immutable Integrity**: On boot, the Rack loads its Passport. It verifies the internal configuration signature against the cluster public key.
*   **Failed scenario**: a scenario that fails to apply stops the gears it started. The Rack keeps the copy of the last scenario it applied successfully. Automatic return to that copy is `[Roadmap]`.

---

## Security roadmap: institutional hardening

To maintain full technical honesty and audit readiness, we distinguish between standard primitives available in the current release and institutional features scheduled for future releases.

### Deterministic masking (planned future)
Unlike heuristic-based masking solutions, **fluxrig** intends to implement **Deterministic Masking** based on the absolute structure of the data:

1.  **SDL Precision**: Fields are tagged as `sensitive` in the Spec Definition Language (SDL).
2.  **Surrogate substitution**: The Rack identifies the sensitive value and swaps it for a masked surrogate on the internal path.
3.  **Stateless Processing Path**: Downstream modules and telemetry sinks only see the surrogate, isolating clear-text data.

> Note: full **tokenization** (a durable, vault-backed value-to-token map, distinct from transient masking) is a separate gear on the roadmap. It is not the same as the parking in the Coat Check (which restores the same value).

### HSM and cloud KMS integration (planned future)
While the current release uses secure file-based keys, the roadmap includes native integration with:

- **Cloud KMS**: AWS KMS and Google Cloud KMS for cluster authority root-of-trust.
- **Hardware Security Modules (HSM)**: Support for PKCS#11 and HashiCorp Vault transit engines.

### Secure execution sandboxing (planned future)
*   **Wasm Logic Gears**: Execution in a sandboxed runtime with no access to host syscalls or networks unless bridged via authorized I/O Gears.
*   **Resource Budgeting**: Enforcement of CPU and memory limits per-Gear.

---

## Security roadmap and compliance

| Feature Area | Status | Implementation Strategy |
| :--- | :--- | :--- |
| **Bus over TLS** | **Available** | The Mixer serves the bus over TLS 1.2 or later. |
| **Client authentication and permissions for each Rack** | **Available (external mode)** | A Rack or a Mixer presents a client certificate or a NATS credentials file to the NATS cluster of the operator. `fluxrig scenario permissions` writes a NATS user for each Rack. The embedded mode has server TLS only. |
| **Sovereign ID** | **Available** | Signed State Envelopes (`rack.flux`). |
| **Field Masking** | **Planned**   | Deterministic PII scrubbers (future). |
| **Cloud KMS** | **Planned**   | AWS/Google KMS integration for Authority keys. |
| **Wasm Execution** | **Available** | Sandboxed execution runtime (wazero), shipped in v0.6.x. |
| **Audit Log persistence** | **Available** | Local CBOR WAL + DuckDB Registry. The storage of what happened; an operational surface over it (console, RBAC) is commercial. |
| **Binary Signing** | **Planned** | Supply chain trust via Sigstore/Cosign. Not built: no release or CI step signs today. |
| **SBOM Generation** | **Planned** | Automated CycloneDX generation per release. |

> [!IMPORTANT]
> **Institutional Compliance**: While **fluxrig** provides the primitives for PCI-DSS and SOC 2 compliance, organizations are responsible for their internal audits. We recommend signing your compiled binaries before production deployment to maintain supply chain integrity.
