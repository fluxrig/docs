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

*   **Secure Tunnel**: All control messages (registry updates, orchestration) and telemetry travel through a persistent, outbound-only **mTLS** connection to the central Mixer.
*   **Data Plane Isolation**: The Rack opens only listening ports explicitly defined by its I/O Gears (for example, a specific TCP socket for protocol ingestion). The Rack isolates these ports strictly from its internal administration and telemetry bus.

### The identity registry
The **Identity Registry** is the foundation of the Unified Control Plane. It maintains the definitive mapping of all Racks, Gears, and Wires in the cluster. Every message therefore carries a cryptographic link to a verified entity.

### The secure mTLS tunnel
The management tunnel uses a **Cryptographic Handshake** that enforces full mutual identity:

- **Identity Minting**: The Mixer acts as the Cluster CA. It issues short-lived, cryptographically-locked certificates to Racks.
- **Mutual Authentication**: Both the Rack and Mixer must present valid Ed25519-backed credentials.
- **Multiplexed Data Streams**: The tunnel acts as a multiplexed pipe. Hundreds of independent logical streams travel over a single physical connection without head-of-line blocking.
- **Bypass Prevention**: The Mixer rejects at the transport layer any message entering without a valid, signed certificate.

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

The retention of a bus stream observes bounds by `snake.stream_max_age` and `snake.stream_max_bytes`.

The setting covers the message store of the bus. It does not encrypt the Mixer analytics database (DuckDB) or its Parquet files. The logs, spans and metrics of the Mixer and the Racks end up there.

Those files underwent a search with a real ISO 8583 authorization: the `19_card_data_iso` test sends one with the card number in DE 2 through two Racks. It reads the database and every Parquet file through DuckDB. It finds the card number in none of them, at the info, debug and trace log levels. The log of the codec names the fields of a message and not their values. The spans of a message carry its identifier and its subject. The search covers the card number of that flow only. It did not search track data, the PIN block or the card verification value.

### The bus in transit

The Mixer can serve the bus over TLS (`snake.tls_cert_file`, `snake.tls_key_file`). It also accepts connections without TLS, and it does not check client certificates. Anyone who can reach the port of the bus can connect and subscribe to every subject. Keep the port on a network that only your Racks and your Mixer can reach. Requiring TLS and authenticating each Rack on the bus is `[Roadmap]`.

### Logs

A gear never writes the payload of a message to a log at INFO or DEBUG. At TRACE it writes the payload, with card numbers written as digits masked to their first six and last four. Use TRACE only for development: the code does not recognize a number in a binary encoding, such as packed BCD, and does not mask it. Never turn TRACE on with real cardholder data.

---

## Sovereign identity (the passport)

A Rack identity does not depend on a live connection to the Mixer: **fluxrig** uses a **Sovereign Identity** model.

*   **The Passport (`state.flux`)**: The Rack does not require a real-time connection to the Mixer to check its own integrity. It holds a signed state bundle (The Passport) on-site.
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
| **mTLS Tunnel** | **Available** | Outbound secure tunnel (TLS 1.2+ Baseline). |
| **Sovereign ID** | **Available** | Signed State Envelopes (`state.flux`). |
| **Field Masking** | **Planned**   | Deterministic PII scrubbers (future). |
| **Cloud KMS** | **Planned**   | AWS/Google KMS integration for Authority keys. |
| **Wasm Execution** | **Available** | Sandboxed execution runtime (wazero), shipped in v0.6.x. |
| **Audit Logging** | **Available** | Local CBOR WAL + DuckDB Registry. |
| **Binary Signing** | **Planned** | Supply chain trust via Sigstore/Cosign. |
| **SBOM Generation** | **Planned** | Automated CycloneDX generation per release. |

> [!IMPORTANT]
> **Institutional Compliance**: While **fluxrig** provides the primitives for PCI-DSS and SOC 2 compliance, organizations are responsible for their internal audits. We recommend signing your compiled binaries before production deployment to maintain supply chain integrity.
