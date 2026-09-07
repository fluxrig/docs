---
slug: /reference/platform/security
title: Security reference
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Security reference

Technical specifications for the **fluxrig** security model, identity envelopes, and transport protocols.

## Identity envelopes (`state.flux`)

The Rack uses a signed CBOR envelope (the "Passport") to maintain its identity across reboots and offline partitions.

### Data structure

```go
// StateEnvelope (The Passport)
type StateEnvelope struct {
    Payload   []byte // CBOR(RackState)
    Signature []byte // Ed25519 Signature
}

// RackState (The Content)
type RackState struct {
    ClusterID      string // e.g. "flux-prod"
    MachineID      string // 128-bit UUID v7 (Registry-assigned)
    Name           string // Human-readable name
    Status         string // active, pending
    Secret         string // Bearer token
    ClusterPublic  []byte // Mixer's public verification key
}
```

---

## Transport: Snake protocol

The Snake tunnel provides the secure mTLS backbone for Rack-to-Mixer communication.

### Default configuration

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **Protocol** | NATS TCP + TLS | Persistent outbound tunnel |
| **Port** | 4222 | Configurable tunnel entry point |
| **Authentication**| mTLS (X.509) | Client & Server certificate exchange |
| **Encryption** | TLS 1.3 / AES-256 | High-entropy session encryption |

---

## Cipher suites

| Purpose | Algorithm | Implementation |
| :--- | :--- | :--- |
| **Signatures** | Ed25519 | Component identity & state |
| **Encryption** | AES-256-GCM | Transport/session (TLS 1.3). Data-at-rest is **[Roadmap]**, see below |
| **Hashing** | BLAKE3 / SHA-256| Integrity checks |
| **IDs** | UUID v7 | Time-sortable unique IDs |

### Data-at-rest encryption

> [!WARNING]
> **At-rest encryption is not yet implemented in fluxrig.** Until it ships, do not persist regulated data (PAN, PIN blocks, track data). Use in-memory ticket storage (RAM only), and run the process hardened by the operator: memory locked against swap (`mlock`) and core dumps disabled. Buffer zeroization on release is [Roadmap] and not performed yet, so redeemed ticket state stays in RAM for its retention window before being dropped.

For persistence, **fluxrig** relies on external or embedded storage engines:

*   **NATS KV state**: the state registry and, when a gear is configured for shared storage, its correlation tickets are stored in NATS JetStream. Correlation state is otherwise **local to the Rack by default**. NATS itself supports native encryption-at-rest in enterprise deployments (a symmetric key managed by the operator, securing the JetStream store files); fluxrig does not yet manage this for you.
*   **WAL logs**: encrypting the Parquet WAL prior to disk flush is a **[Roadmap]** feature.

---

## Wasm Component PKI

The execution of third-party Wasm logic at the edge necessitates strict supply chain security. `fluxrig` utilizes a **Dual-Signature PKI model** for all Wasm modules:

1. **Vendor Roots**: The Mixer maintains a directory of trusted Vendor Ed25519 Public Keys (`data/wasm/keys`). 
2. **Module Signature**: Third-party developers sign their `.wasm` payloads with their private key, embedding a `fluxrig.signature` directly into the Wasm custom sections.
3. **Mixer Verification & Countersignature**: During `fluxrig wasm import`, the Mixer cryptographically verifies the vendor signature. If valid, the Mixer applies its own Cluster Authority signature (`fluxrig.cluster.signature`) to the module and publishes it to the registry.
4. **Rack Execution Guard**: Racks download the Wasm modules via the mTLS NATS Snake. Before JIT compilation via `wazero`, the Rack validates the Mixer's countersignature. Any module lacking a valid signature from the trusted Cluster Authority is immediately dropped and a critical security alert is dispatched.

---

## Key management CLI

| Command | Purpose | Access |
| :--- | :--- | :--- |
| `fluxrig keys gen-cluster` | Generate root cluster keys | Mixer Admin |
| `fluxrig keys gen-client` | Generate mTLS client certs | Mixer Admin |
| `fluxrig admin enroll` | Initiate Rack enrollment | Physical Access |

---

## API authentication & management

The Mixer REST API is secured via two mechanisms:

1. **mTLS (Internal)**: Administrative CLI commands (`fluxrig admin`) executed on the local network use mTLS to authenticate against the Mixer.
2. **Bearer Tokens (External)**: For integrations with CI/CD or Enterprise Web Dashboards, the Mixer requires a signed JWT Bearer token configured at bootstrapping.

### Certificate rotation
> [!WARNING]
> **Planned Feature**: Zero-downtime certificate rotation is currently on the roadmap.

Currently, when `fluxrig keys gen-cluster` generates new trust roots, the Mixer and Racks must be restarted to transition to the new Root CA. Future releases will allow the Mixer to advertise the impending rotation, allowing Racks to automatically transition without breaking ongoing data plane traffic.
