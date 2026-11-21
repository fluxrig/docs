---
slug: /reference/platform/security
title: Security reference
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Security reference

Technical specifications for the **fluxrig** security model, identity envelopes, and transport protocols.

## Identity envelopes (`rack.flux`)

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

The Snake carries all Rack-to-Mixer communication.

### Default configuration

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **Protocol** | NATS TCP + TLS | Persistent outbound tunnel |
| **Port** | 4222 | Configurable tunnel entry point |
| **Authentication**| Server certificate (X.509) | The Rack verifies the server. In the embedded mode the bus does not authenticate the Rack. In the external mode the NATS cluster of the operator can: see [Rack identity](../architecture/security.md#rack-identity) |
| **Encryption** | TLS 1.2 or later | The version and the cipher are negotiated |

### Client settings

A Mixer and a Rack connect to the bus with the same `[snake]` settings:

| Setting | Mode | Description |
| :--- | :--- | :--- |
| `root_ca_file` | Both | The CA that verifies the server certificate. |
| `client_cert_file`, `client_key_file` | External | The certificate that this client presents to a [NATS](https://docs.nats.io) server that asks for one. Set both or neither. |
| `credentials_file` | External | A NATS credentials file: the user of this client, with the permissions that the operator gave it. |

The client checks that each file exists when it starts. A path that it cannot read stops the start with its own error.

In the embedded mode the server has no users, and the Mixer connects to its own server with no client certificate. Client certificates and NATS users need the external mode (`snake.external = true`).

### NATS users for each Rack

`fluxrig scenario permissions` reads one or more scenario files and writes one NATS user for each Rack. Put the users in the `authorization { users: [...] }` block of the external NATS, next to the users of the Mixers. The command writes no user for a Mixer.

```bash
fluxrig scenario permissions scenario.yaml --rack rack-a --rack rack-b \
  --user 'CN={rack},O=example' --config rack.toml
```

- `--user` is the NATS user of a Rack, with `{rack}` for its name. With `verify_and_map`, it is the subject of the certificate of the Rack.
- `--config` reads the subject names from a Rack configuration: `snake.stream_name`, `telemetry.base_subject`, `telemetry.stream_name` and, when `snake.dlq_enabled` is set, `snake.dlq_subject`.
- With several scenarios, each Rack gets what all of them need, so it can run any one of them.

The flags are in the [CLI reference](cli.md#scenario-management-simulation).

Each user allows `publish` and `subscribe` on these subjects, and sets `allow_responses: true`:

| Direction | Subjects |
| :--- | :--- |
| Publish | `flux.agent.hello`, `flux.agent.heartbeat`, `flux.msg.<rack>.>`, the connection events of the Rack, the telemetry subjects, and the dead letter subjects when the Rack uses them |
| Publish | The subject of each wire that the Rack consumes, for the sync probe |
| Publish | The JetStream API calls of the Rack: consumers on its enrollment, notification and scenario subjects, on the wires it consumes and on its telemetry handshake, and the acknowledgements on its streams |
| Publish | What the manifest of each gear declares: the connection events it reads, and the key-value buckets it reads, writes or creates |
| Publish | `flux.ctrl.link.<gear>`, the link state of each gear of the Rack |
| Subscribe | `_INBOX_<rack>.>`, the reply inbox of the Rack |
| Subscribe | `flux.ctrl.<gear>` for each gear of the Rack, and `flux.ctrl.link.<gear>` of each I/O gear that a gear of the Rack is bound to |

A gear declares what it does on the bus besides its wires in `BusNeeds` of its manifest. A gear type that declares nothing gets only what every gear needs.

The Racks of a scenario are the Racks that its gears are deployed to. A gear that names no Rack runs on every Rack, so the command needs `--rack` for each Rack. Generate the users again when a scenario changes. A Rack that lacks a permission logs the refusal at ERROR.

What these permissions do not refuse is in [Permissions for each Rack](../architecture/security.md#permissions-for-each-rack).

Forwarded control writes use `flux.mixer.control.>`. The Mixer that holds the write lease answers each request. These requests carry no caller identity or credentials. NATS subject authorization is the only access fence. Any NATS user with access to this subject can issue or intercept a write.

Do not grant Rack users publish or subscribe access to `flux.mixer.control.>`. `fluxrig scenario permissions` does not grant this access. Do not grant a Rack `flux.>` or `flux.mixer.>`.

---

## Cipher suites

| Purpose | Algorithm | Implementation |
| :--- | :--- | :--- |
| **Signatures** | Ed25519 | Component identity & state |
| **Encryption** | ChaCha20-Poly1305 (default) or AES-256-GCM | Data at rest (the bus message store). TLS 1.3 for transport/session. |
| **Hashing** | BLAKE3 / SHA-256| Integrity checks |
| **IDs** | UUID v7 | Time-sortable unique IDs |

### Data-at-rest encryption

In the embedded mode, the Mixer's message store (every message on a guaranteed wire, plus its key-value buckets) is encrypted on disk by default. In the external mode the store belongs to the NATS cluster of the operator. See [data at rest](../architecture/security.md#data-at-rest-and-in-logs) for the full mechanism, key management, and rotation.

Two stores are not covered by this setting:

*   **The Mixer's analytics database (DuckDB) and Parquet files**, where logs, spans and metrics end up, are not encrypted by it. Regulated data (PAN, PIN blocks, track data) is not written there in the first place; see [data at rest](../architecture/security.md#data-at-rest-and-in-logs) for what was actually searched for.
*   **A ticket store using `memory`** (the [Conductor gear](gears/conductor.md#ticket-store-strategy)'s default) keeps its state in the Mixer process's RAM, never on disk. RAM-only is not automatically safe: for regulated data, the process still needs operator hardening (memory locked against swap with `mlock`, core dumps disabled). Buffer zeroization on release is `[Roadmap]` and not performed yet, so a redeemed ticket's state stays in RAM for its retention window before being dropped. A ticket store using `local_durable` or `shared` is not encrypted and must not hold regulated data until that ships.

---

## Wasm Component PKI

The execution of third-party Wasm logic at the edge necessitates strict supply chain security. `fluxrig` utilizes a **Dual-Signature PKI model** for all Wasm modules:

1. **Vendor Roots**: The Mixers trust the vendor Ed25519 public keys of one keyring on the bus. See [Trusted vendor keys](#trusted-vendor-keys).
2. **Module Signature**: Third-party developers sign their `.wasm` payloads with their private key, embedding a `fluxrig.signature` directly into the Wasm custom sections.
3. **Mixer Verification & Countersignature**: During `fluxrig wasm import`, the Mixer cryptographically verifies the vendor signature. If valid, the Mixer applies its own Cluster Authority signature (`fluxrig.cluster.signature`) to the module and stores it in the bucket `wasm_catalog` of the bus. When the bus does not take the module, the import answers 503 and imports nothing.
4. **Rack Execution Guard**: Racks download the Wasm modules through the Snake. Before JIT compilation via `wazero`, the Rack validates the Mixer's countersignature. A module from the catalog without a valid signature from the Cluster Authority does not run, and the gear fails to start with the reason.

### Trusted vendor keys

The trusted vendor keys are one keyring in the bucket `wasm_keys` of the bus, with `snake.replicas` copies. Every Mixer reads the keyring at each import. Thus a vendor that is trusted or revoked through one Mixer counts on every Mixer at its next import. The signed modules are in the bucket `wasm_catalog`, so every Mixer lists the same modules.

**The keys directory.** At startup a Mixer adds to the keyring each key of `wasm.trusted_keys_dir` (default `./data/wasm/keys`) that the keyring does not hold:

- A key file is `<vendor>.pub`, with the 32 bytes of an Ed25519 public key.
- The vendor name must match `^[A-Za-z0-9_-]{1,64}$`. The Mixer ignores a file with another name, or with a key of another size, and logs a Warn. A module signed with that key is then refused at import, unless the import allows unsigned modules.
- The keyring wins. A key that the keyring holds with other bytes, or holds as revoked, does not change, and the Mixer reports the file with a Warn.

Deleting a key file does not stop trust in its vendor. Revoke the vendor.

**The API.** The keyring routes need the API token, as every route but `/api/v1/health`:

| Request | Effect | Answers |
| :--- | :--- | :--- |
| `GET /api/v1/wasm/keys` | Lists the vendors, with the SHA-256 fingerprint of each key, revoked ones included | `200`, `503` |
| `PUT /api/v1/wasm/keys/{vendor}` | Trusts the Ed25519 public key in the body, 32 raw bytes, or replaces it. A revoked vendor is trusted again | `204`, `400` for a vendor name or a key that is not valid, `503` |
| `DELETE /api/v1/wasm/keys/{vendor}` | Revokes the vendor | `204`, `400`, `503` |

`503` means that the keyring on the bus does not answer. Try the request again.

A revoked vendor stays in the keyring as revoked. Thus a key file that a Mixer still holds does not add it back.

Revocation applies to imports. The Mixer refuses a new import that the revoked key signed, unless the import allows unsigned modules. A module that is already in the catalog keeps its cluster signature, and the Racks still run it. The catalog lists it as `Untrusted`.

**The CLI.** `fluxrig wasm keys list`, `fluxrig wasm keys trust <vendor> <key-file>` and `fluxrig wasm keys revoke <vendor>` call these routes. See the [CLI reference](cli.md#wasm-catalog-operations).

---

## Key management CLI

| Command | Purpose | Access |
| :--- | :--- | :--- |
| `fluxrig keys gen-cluster` | Generate root cluster keys | Mixer Admin |
| `fluxrig admin racks approve` | Approve a Rack that asked to enroll | Mixer Admin |
| `fluxrig wasm keys trust <vendor> <key-file>` | Trust a Wasm vendor key on every Mixer | Mixer Admin |
| `fluxrig wasm keys revoke <vendor>` | Stop trusting a Wasm vendor on every Mixer | Mixer Admin |

---

## API authentication & management

The Mixer REST API is secured by one shared bearer token, `api.auth_token`, required as `Authorization: Bearer <token>` on every route except `/api/v1/health`. The comparison is constant-time. The Mixer refuses to start with no token configured, unless `api.auth_disabled_dangerously` is set explicitly.

This is the same token for every caller: the `fluxrig admin`, `fluxrig configuration`, `fluxrig logs`, `fluxrig metrics`, `fluxrig racks`, `fluxrig topology`, and `fluxrig scenario --api` commands all send it via `--api-token` (or `FLUXRIG_API_TOKEN`). There is no separate mTLS path for the CLI, and the token is not a JWT: it is an opaque, operator-chosen secret.

TLS also protects the Snake transport between a Rack and the bus (see [Transport: Snake protocol](#transport-snake-protocol) above), which is a different connection from the REST API this section covers.

### Certificate rotation
> [!WARNING]
> **Planned Feature**: Zero-downtime certificate rotation is currently on the roadmap.

Currently, when `fluxrig keys gen-cluster` generates new trust roots, the Mixer and Racks must be restarted to transition to the new Root CA. Future releases will allow the Mixer to advertise the impending rotation, allowing Racks to automatically transition without breaking ongoing data plane traffic.
