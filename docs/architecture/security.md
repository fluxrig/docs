---
title: Security architecture
slug: /architecture/security
---

# Security architecture

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

**fluxrig** is engineered for the most hostile network environments (Zero Trust) and the most sensitive data workloads (e.g., PCI-DSS, HIPAA). Our security model relies on **Network Isolation** and **Sovereign Identity**, ensuring that the integrity of the transactional processing path is never compromised.

## Network isolation (Inbound Zero)

To protect infrastructure from external threats, we implement a strict **"Inbound Zero"** policy for the management plane.

### Invisible infrastructure
By moving all orchestration and management logic to an outbound-only tunnel, **fluxrig** removes standard vectors (SSH, HTTP, SNMP) as public entry points. This dramatically reduces the network attack surface, making the Rack effectively invisible to external scans.

*   **Secure Tunnel**: All control messages (registry updates, orchestration) and telemetry travel via a persistent, outbound-only **mTLS** connection to the central Mixer.
*   **Data Plane Isolation**: The Rack only opens listening ports explicitly defined by its I/O Gears (e.g., a specific TCP socket for protocol ingestion). These ports are strictly isolated from the Rack's internal administration and telemetry bus.

### The Identity Registry
The **Identity Registry** is the foundation of the Unified Control Plane. It maintains the definitive mapping of all Racks, Gears, and Wires in the cluster, ensuring that every message is cryptographically tethered to a verified entity.

### The secure mTLS tunnel
The management tunnel uses a **Cryptographic Handshake** that enforces 100% mutual identity:

- **Identity Minting**: The Mixer functions as the Cluster CA, issuing short-lived, cryptographically-locked certificates to Racks.
- **Mutual Authentication**: Both the Rack and Mixer must provide valid Ed25519-backed credentials.
- **Multiplexed Data Streams**: The tunnel functions as a multiplexed pipe, allowing hundreds of independent logical streams to travel over a single physical connection without head-of-line blocking.
- **Bypass Prevention**: Any message entering the Mixer without a valid, signed certificate is rejected at the transport layer.

---

## Sovereign identity (The Passport)

To ensure operational continuity during network partitions, **fluxrig** utilizes a **Sovereign Identity** model.

*   **The Passport (`state.flux`)**: The Rack does not require a real-time connection to the Mixer to verify its own integrity. It holds a signed state bundle (The Passport) on-site.
*   **Cluster Authority**: The root of trust is the **Cluster Authority Key**. In the current release, this is a **file-based Ed25519 keypair**.
*   **Immutable Integrity**: On boot, the Rack loads its Passport and verifies the internal configuration signature against the cluster public key.
*   **Safe Rollback**: If a new configuration from the Mixer fails a signature check or causes a bootstrap error, the Rack automatically rolls back to the previous known-good state in its local storage.

---

## Security roadmap: institutional hardening

To maintain 100% technical honesty and audit readiness, we distinguish between standard primitives available in the current release and institutional features scheduled for future releases.

### Deterministic masking (Planned future)
Unlike heuristic-based masking solutions, **fluxrig** intends to implement **Deterministic Masking** based on the absolute structure of the data:

1.  **SDL Precision**: Fields are tagged as `sensitive` in the Spec Definition Language (SDL).
2.  **Surrogate substitution**: The Rack identifies the sensitive value and swaps it for a masked surrogate on the internal path.
3.  **Stateless Processing Path**: Downstream modules and telemetry sinks only see the surrogate, isolating clear-text data.

> Note: full **tokenization** (a durable, vault-backed value↔token map, distinct from transient masking) is a separate gear on the roadmap; it is not the same as the Coat Check's parking (which restores the same value).

### HSM and Cloud KMS integration (Planned future)
While the current release uses secure file-based keys, the roadmap includes native integration with:

- **Cloud KMS**: AWS KMS and Google Cloud KMS for cluster authority root-of-trust.
- **Hardware Security Modules (HSM)**: Support for PKCS#11 and HashiCorp Vault transit engines.

### Secure execution sandboxing (Planned future)
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