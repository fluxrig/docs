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
| **Encryption** | ChaCha20-Poly1305 (default) or AES-256-GCM | Data at rest (the bus message store). TLS 1.3 for transport/session. |
| **Hashing** | BLAKE3 / SHA-256| Integrity checks |
| **IDs** | UUID v7 | Time-sortable unique IDs |

### Data-at-rest encryption

The Mixer's message store (every message on a guaranteed wire, plus its key-value buckets) is encrypted on disk by default. See [data at rest](../architecture/security.md#data-at-rest-and-in-logs) for the full mechanism, key management, and rotation.

Two stores are not covered by this setting:

*   **The Mixer's analytics database (DuckDB) and Parquet files**, where logs, spans and metrics end up, are not encrypted by it. Regulated data (PAN, PIN blocks, track data) is not written there in the first place; see [data at rest](../architecture/security.md#data-at-rest-and-in-logs) for what was actually searched for.
*   **A ticket store using `memory`** (the [Conductor gear](gears/conductor.md#ticket-store-strategy)'s default) keeps its state in the Mixer process's RAM, never on disk. RAM-only is not automatically safe: for regulated data, the process still needs operator hardening (memory locked against swap with `mlock`, core dumps disabled). Buffer zeroization on release is `[Roadmap]` and not performed yet, so a redeemed ticket's state stays in RAM for its retention window before being dropped. A ticket store using `local_durable` or `shared` is not encrypted and must not hold regulated data until that ships.

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

The Mixer REST API is secured by one shared bearer token, `api.auth_token`, required as `Authorization: Bearer <token>` on every route except `/api/v1/health`. The comparison is constant-time. The Mixer refuses to start with no token configured, unless `api.auth_disabled_dangerously` is set explicitly.

This is the same token for every caller: the `fluxrig admin`, `fluxrig configuration`, `fluxrig logs`, `fluxrig metrics`, `fluxrig racks`, `fluxrig topology`, and `fluxrig scenario --api` commands all send it via `--api-token` (or `FLUXRIG_API_TOKEN`). There is no separate mTLS path for the CLI, and the token is not a JWT: it is an opaque, operator-chosen secret.

mTLS is used elsewhere in `fluxrig`, for the Snake transport between a Rack and the Mixer (see [Transport: Snake protocol](#transport-snake-protocol) above), which is a different connection from the REST API this section covers.

### Certificate rotation
> [!WARNING]
> **Planned Feature**: Zero-downtime certificate rotation is currently on the roadmap.

Currently, when `fluxrig keys gen-cluster` generates new trust roots, the Mixer and Racks must be restarted to transition to the new Root CA. Future releases will allow the Mixer to advertise the impending rotation, allowing Racks to automatically transition without breaking ongoing data plane traffic.
