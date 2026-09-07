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
├── blobs/             # SHA-256 content blobs (sharded)
│   ├── a1/
│   │   └── a1b2c3...
│   └── d4/
│       └── d4e5f6...
```

- **CAS store**: Each artifact (spec or scenario YAML) is hashed with SHA-256 and stored as an immutable blob. The hash is the address: the same bytes always resolve to the same blob, and changed bytes are a different artefact rather than a new version of the same one.
- **Index**: A JSON file mapping logical `name → tag → hash`, supporting both Specs and Scenarios.

## URN scheme

Artifacts are referenced using the short `name:tag` format:

| Format | Example | Resolution |
| :--- | :--- | :--- |
| `name:tag` | `visa:v1.0.0` | Look up hash in index under name + tag. |
| `name:latest` | `visa:latest` | Resolve to the highest SemVer tag for that name. |
| `sha256:hash` | `sha256:a1b2c3...` | Direct CAS blob lookup. |

> [!NOTE]
> **Resolution Timing**: URNs are resolved **on boot or reload**. If a gear is configured with `visa:latest`, it grabs the newest version available at the moment it starts. Importing a newer version into the store later does *not* automatically hot-swap the active gear; a scenario restart is required.

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

If `--name` and `--tag` are omitted, they come from the document itself: a spec
declares `spec.id` and `spec.version`, a scenario declares `meta.name` and
`meta.version`. A spec's `spec.name` is its human title and is not used as the
reference: "ISO 8583:1987 (ASCII)" is not something to put in a `name:tag`. A version written as `2.2.0` is filed as `v2.2.0`. Only a
document that declares no version at all falls back to auto-incrementing the
minor from the latest existing tag.

Importing identical content again is idempotent: the same bytes stay one
artefact under one tag. A document claiming a version that already names
different content is **refused**, because a version identifies one set of bytes
or it identifies nothing. That is what makes a tag usable as a deployment
reference.

### List and history

```bash
fluxrig spec list                        # every spec the store holds
fluxrig spec list --json                 # the same, for a script
fluxrig spec history iso8583-v87-ascii   # every version of one spec
fluxrig scenario list
```

A listing carries what the store recorded at import: when each version was filed,
its size, the document's human title, and which version a reference without a tag
resolves to.

```
NAME               VERSION          IMPORTED              SIZE     HASH          TITLE
iso8583-v87-ascii  v2.2.0 (latest)  2026-09-06 18:47 UTC  33.6 KB  256e09885b96  ISO 8583:1987 (ASCII)
```

A history is ordered by version, newest first, and **not** by arrival: a patch to
an older branch is imported after a newer release without being newer than it.

An artefact filed before the store recorded dates shows `unrecorded` rather than
a guess.

### A spec states a contract

A spec must declare `spec.id` and `spec.version`, and a spec missing either is
refused when it loads: at boot, with a message naming the spec, never per
transaction.

Every message carries three stamps, and they answer different questions:

| Metadata | Answers |
| :--- | :--- |
| `codec.spec_hash` | Which bytes ran. Changes when any character does. |
| `codec.spec_id` | Which spec. A Rack may run several. |
| `codec.spec_version` | Which contract. |

## Reading a stored spec

The Mixer serves what its store holds, including the protocol reference rendered
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

The reference is derived on each request rather than stored, so it cannot fall
behind the spec. The same document is rendered locally by
[`fluxrig spec doc`](cli.md).

## Mixer integration

The Mixer uses the manager at startup to resolve `name:tag` scenario references (see [Scenario Reference](scenario.md#startup-resolution)). The resolution flow:

1. Parse the reference string.
2. If it contains `/` → treat as file path, read from disk.
3. If empty → resume last active scenario from the store.
4. Otherwise → open `data/store/`, call `manager.Load(urn)` to resolve `name:tag` from the CAS.

## How a scenario names a spec

A scenario names specs per gear, in the gear's own configuration. There is no
scenario-wide spec:

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
own store before applying the scenario, because a gear resolves its spec while
initialising. Two Racks given one scenario therefore compile the same bytes, which a
path cannot promise that, since it resolves against whatever each Rack happens to
hold at that location.

If the Mixer cannot resolve something a scenario names, it warns and deploys
anyway. The Rack then reports the unresolvable reference by the name the scenario
actually used, which is more use than a deployment refused over a spec only the
Mixer could not find.