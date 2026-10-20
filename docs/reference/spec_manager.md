---
slug: /reference/protocol/spec-manager
title: Spec & scenario manager
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Spec & scenario manager

The **Spec & Scenario Manager** provides a local, Git-friendly registry for Specs and Scenarios. It uses a Content-Addressable Store (CAS) to ensure every version is immutable, auditable, and reproducible.

## The content-addressable store {#cas}

```
data/store/
├── index.json         # name → tag → hash mapping
├── index.lock         # serializes concurrent index writers
├── blobs/             # SHA-256 content blobs (sharded)
│   ├── a1/
│   │   └── a1b2c3...
│   └── d4/
│       └── d4e5f6...
```

- **CAS store**: The system hashes each artifact (spec or scenario YAML) with SHA-256 and stores it as an immutable blob. The hash is the address. The same bytes always resolve to the same blob. Changed bytes are a different artefact rather than a new version of the same one.
- **Index**: A JSON file maps logical `name → tag → hash`. It supports both Specs and Scenarios.
- **Concurrent writers**: A CLI and a Mixer can import at the same time against one store. An import takes a lock on `index.lock`, re-reads the index, merges its own entry into what is on disk, and replaces the file atomically. Two imports therefore both keep their versions, and a reader never sees a half-written index.
- **A running Mixer reads its index when it starts.** It does not see a version that a CLI imports into its store directory later. To add an artefact to a running Mixer, [upload it through the API](#upload).

## URN scheme

Reference artifacts with the short `name:tag` format:

| Format | Example | Resolution |
| :--- | :--- | :--- |
| `name:tag` | `visa:v1.0.0` | Look up hash in index under name + tag. |
| `name:latest` | `visa:latest` | Resolve to the highest SemVer tag for that name. |
| `sha256:hash` | `sha256:a1b2c3...` | Direct CAS blob lookup. |

> [!NOTE]
> **Resolution timing**: The system resolves URNs **on boot or reload**. If you configure a gear with `visa:latest`, it grabs the newest version available at the moment it starts. Importing a newer version into the store later does *not* automatically hot-swap the active gear. You need a scenario restart.

> [!NOTE]
> Tags must follow [Semantic Versioning](https://semver.org) (e.g., `v1.0.0`, `v2.1.3`).

## CLI commands

### Import

Snapshot a local file into the CAS:

```bash
# Import a spec
fluxrig spec import card_schema.yaml --name visa --tag v1.0.0

# Import a scenario
fluxrig scenario import payment_flow.yaml --name payment-flow --tag v1.0.0
```

If you omit `--name` and `--tag`, they come from the document itself: a spec
declares `spec.id` and `spec.version`, and a scenario declares `meta.name` and
`meta.version`. A spec's `spec.name` is its human title, and the system does not use it as the
reference: do not put "ISO 8583:1987 (ASCII)" in a `name:tag`. The system files a version written as `2.2.0` as `v2.2.0`. Only a
document that declares no version at all causes the system to auto-increment the
minor from the latest existing tag.

Importing identical content again is idempotent: the same bytes stay one
artefact under one tag. The system **refuses** a document that claims a version which already names
different content. A version identifies one set of bytes, or it identifies nothing. That is what makes a tag usable as a deployment
reference.

### List and history

```bash
fluxrig spec list                        # every spec the store holds
fluxrig spec list --json                 # the same, for a script
fluxrig spec history iso8583-v87-ascii   # every version of one spec
fluxrig scenario list
```

A listing carries what the store recorded at import. It shows when the system filed each version. It shows
its size. It shows the document's human title. It shows which version a reference without a tag
resolves to.

```
NAME               VERSION          IMPORTED              SIZE     HASH          TITLE
iso8583-v87-ascii  v3.0.0 (latest)  2026-10-20 14:58 UTC  32.4 KB  0f4aa6aad084  ISO 8583:1987 (ASCII)
```

The system orders a history by version, newest first, and **not** by arrival. It imports
a patch to an older branch after a newer release without making it newer than the release.

An artefact that the system filed before the store recorded dates shows `unrecorded` rather than
a guess.

### A spec states a contract

A spec must declare `spec.id` and `spec.version`. The system refuses a spec that misses either when it loads: at boot, with a message naming the spec, never per
transaction.

Every message carries three stamps, and they answer different questions:

| Metadata | Answers |
| :--- | :--- |
| `codec.spec_hash` | Which bytes ran. Changes when any character does. |
| `codec.spec_id` | Which spec. A Rack may run several. |
| `codec.spec_version` | Which contract. |

## Reading a stored spec

The Mixer serves what its store holds, including the protocol reference that it renders
from it:

```
GET /api/v1/specs                       # every spec, with its attributes
GET /api/v1/specs/{name}                # every version of one spec, newest first
GET /api/v1/specs/{name}/{tag}          # the document, byte for byte
GET /api/v1/specs/{name}/{tag}/doc      # the protocol reference
```

The reference takes `?scope=public|complete` and `?format=html|markdown`. It
defaults to **public**, which omits fields the spec marks `scope: private`,
because the endpoint carries no authentication of its own.

The Mixer derives the reference on each request rather than storing it, so it cannot fall
behind the spec. [`fluxrig spec doc`](cli.md) renders the same document locally.

## Uploading to the Mixer {#upload}

The Mixer sends with a scenario the artefacts that the scenario names by store reference, so that each Rack resolves them from its own store. Upload an artefact to the Mixer before you activate a scenario that names it:

```bash
export FLUXRIG_API_TOKEN=<api.auth_token>
fluxrig spec import --api specs/iso8583-v87-ascii.yaml
fluxrig spec import --api --name billers --tag v1 lookups/billers.yaml
```

The command sends the file to `POST /api/v1/specs`. `--api-url` (or `FLUXRIG_API_URL`) names the Mixer, and the default is `http://localhost:8090`. `--api-timeout` sets how long the command waits for the answer (default `30s`).

The Mixer finds the kind of the artefact from its header and checks it with the loader of that kind. It stores only an artefact that a Rack can load:

| Kind | How the Mixer finds it | What the Mixer checks |
| :--- | :--- | :--- |
| `iso8583` | A `spec:` header, with no other kind | The [ISO8583 SDL](specs/iso8583_sdl.md) loader. |
| `iso20022` | `spec.wire.source` names an ISO 20022 wire source | The [ISO 20022 spec](specs/iso20022_spec.md) loader. |
| `translation` | `spec.kind: translation` | The [translation spec](gears/translate.md#the-translation-spec) loader, against the store. Each protocol spec and lookup source that the translation names must be a store reference that the store already holds. |
| `data` | No `spec:` header | A YAML map of records, for a [lookup](gears/translate.md#lookups). Data declares no name or version, so `--name` and `--tag` are mandatory. |

Upload in this order: the protocol specs and the lookup data first, then the translation specs that name them.

A translation spec names further artefacts: its protocol specs and its lookup records. The Mixer sends these with the scenario too, so a Rack can run a translation with none of its files.

| Answer | Meaning |
| :--- | :--- |
| `201` | Stored. The body gives the `urn`, the `hash` and the `kind`. Identical content uploaded again also gives `201`. |
| `400` | The artefact does not load. The body names what is wrong. |
| `409` | The `name:tag` already names other content. Change the version. |
| `413` | The artefact is larger than [`mixer.max_payload_size`](configuration.md). An artefact travels to the Racks inside the scenario payload, so it cannot be larger than a payload. |

Then import the scenario and activate it, which deploys it to its Racks:

```bash
fluxrig scenario import --api --activate scenarios/billpay_rtp.yaml
```

## Mixer integration

The Mixer uses the manager at startup to resolve `name:tag` scenario references (see [Scenario Reference](scenario.md#startup-resolution)). The resolution flow:

1. Parse the reference string.
2. If it contains `/`, treat it as a file path. Read it from disk.
3. If it is empty, resume the last active scenario from the store.
4. Otherwise, open `data/store/`. Call `manager.Load(urn)` to resolve `name:tag` from the CAS.

## How a scenario names a spec

A scenario names specs per gear, in the gear's own configuration. No scenario-wide spec exists:

```yaml
gears:
- name: codec-in
  type: codec_iso8583
  config:
    spec_path: "visa:v1.0.0"    # a store reference; a path also works
    direction: decode
```

At deploy time the Mixer reads those references, resolves them against its own
store, and sends the artefacts **with** the scenario. The Rack files them in its
own store before it applies the scenario, because a gear resolves its spec while it initialises. Two Racks that receive one scenario therefore compile the same bytes. A path cannot promise this, since it resolves against whatever each Rack happens to hold at that location.

If the Mixer cannot resolve something a scenario names, it warns and deploys
anyway. The Rack then reports the unresolvable reference by the name the scenario
actually used. This is more use than a deployment that the Mixer refused over a spec it could not find.