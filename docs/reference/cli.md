---
slug: /reference/operations/cli
title: CLI reference
---

# CLI reference

The **Command Line Interface (CLI)** is the primary orchestration tool for the **fluxrig** ecosystem. It provides a unified, static binary interface for managing the entire lifecycle of distributed infrastructure (from key generation and node registration to real-time telemetry exploration).

## fluxrig (static binary)
The unified tool for operators and edge runtimes.

### Core commands

| Command | Description |
|---------|-------------|
| `fluxrig` | Root command, displays help |
| `fluxrig version` | Display version information |
| `fluxrig rack` | Start the Rack edge node |
| `fluxrig racks` | List the Racks the Mixer has registered |
| `fluxrig run` | Starts the same Rack runtime as `rack`; both call `RunAgent` |

### Key management

| Command | Description |
|---------|-------------|
| `fluxrig keys gen-cluster` | Generate a new Cluster Authority keypair |
| `fluxrig keys inspect <path>` | Inspect a state.flux Passport file |

**Flags for `gen-cluster`**:

- `-d, --dir` - Output directory (default: `.`)
- `-n, --name` - Filename (default: `cluster.key`)
- `-o, --out` - Full output path (overrides dir/name)

### Wasm catalog operations

Wasm payloads use cryptographic identities to ensure secure edge distribution.

| Command | Description |
|---------|-------------|
| `fluxrig wasm sign <file>` | Sign a `.wasm` file using an Ed25519 vendor key |
| `fluxrig wasm import <file>` | Import and validate a signed Wasm binary into the Mixer catalog |

**Usage**:
```bash
# Developer: sign the compiled logic
fluxrig wasm sign my_logic.wasm --key ./vendor.key

# Operator: import the logic into the sovereign catalog
fluxrig wasm import my_logic.wasm
```

### Administration

| Command | Description |
|---------|-------------|
| `fluxrig admin racks list` | List all registered racks |
| `fluxrig admin racks list --status pending` | [DEPRECATED] Use API for status filtering. |
| `fluxrig admin racks approve <id> --name <name>` | Approve a pending rack and assign identity |
| `fluxrig admin racks suspend <id>` | Suspend a rack (blocks traffic, keeps identity) |
| `fluxrig admin racks activate <id>` | Reactivate a suspended rack |
| `fluxrig admin racks remove <id>` | Remove a rack from registry (destroys identity) |
| `fluxrig admin racks shutdown <id>` | Gracefully shutdown a Rack edge node |
| `fluxrig admin racks set-log-level <id> <level>` | Set log level (debug/info/warn/error) |

**Common Flag**:

- `--api-url` - Mixer API URL (default: `http://localhost:8090`)

### Spec management (registry)

Specs are governed via a [Content-Addressable Store (CAS)](spec_manager.md#cas).

| Command | Description |
|---------|-------------|
| **`import <file>`** | Snapshot a local YAML spec into the CAS. |
| **`list`** | Every spec the store holds, with when each version was filed, its size and its title. |
| **`history <name>`** | Every version of one spec, newest first by version rather than by arrival. |
| **`export <urn> <file>`** | Write a CAS spec back to a local file. |
| **`doc <file>`** | Render the protocol reference from a spec. |

**Usage**:
```bash
fluxrig spec import visa_spec.yaml --name visa --tag v1.0.0
fluxrig spec list --json
fluxrig spec history iso8583-v87-ascii
```

**Flags for `import`**:

- `--name` - Logical name (e.g., `visa`). Omitted, it comes from `spec.id`.
- `--tag` - Semantic version (e.g., `v1.0.0`). Omitted, it comes from `spec.version`.
- `--store-dir` - CAS store location (default: `~/.fluxrig/store`)

**Flags for `list` and `history`**:

- `--json` - Machine-readable output.

#### Rendering the protocol reference

`fluxrig spec doc` renders what a spec says about its protocol: the messages,
what each one carries, and what every data element means. It is derived on each
render, so it cannot fall behind the spec.

```bash
fluxrig spec doc examples/specs/iso8583-v87-ascii.yaml --format html --out reference.html
```

- `--format` - `markdown` (default) for a repository or a docs site, `html` for a
  page that is read, printed or saved as a PDF. The HTML is self-contained: no
  scripts, stylesheets or fonts are fetched, so it works offline.
- `--scope` - `public` (default) omits fields the spec marks `scope: private` and
  says how many it withheld; `complete` carries everything. Only the public
  variant is eligible for publication.
- `--out` - Output file. Defaults to stdout.
- `--title` - Heading. Defaults to the spec's own name.

The spec is loaded before it is rendered, so one that does not resolve fails
here.

[Protocol reference](./specs/protocol_reference.md) explains what the document
contains and what a spec has to say for it to be worth reading, and links a
rendered example.

The Mixer serves the same document for a spec in its store. See
[Spec manager](spec_manager.md#reading-a-stored-spec).

### Gear catalog

| Command | Description |
|---------|-------------|
| **`gears list`** | Every registered gear type, with its category and status. |
| **`gears doc [type]`** | The manifest as Markdown: identity, ports and configuration fields. `--all` prints one section per gear; `--write <dir>` refreshes the reference pages. |
| **`gears manifest [type]`** | The same manifest, machine-readable. This is what scenario validation and tooling read. |

### Scenario management (simulation)

Scenarios define the operational topology used for testing and simulation. They can be imported into the CAS or executed directly as a standalone test runner.

| Command | Description |
|---------|-------------|
| **`import <file>`** | Snapshot a scenario YAML into the CAS. |
| **`list`** | List all stored scenarios. |
| **`export <urn> <file>`** | Export a CAS scenario for local editing. |
| **`diff <file>`** | Compare local scenario file with the active one. |

**Usage**:
```bash
# Import into CAS
fluxrig scenario import main.yaml --name dev --tag v1.0.0

# Hot-Reload via Mixer API
fluxrig scenario import main.yaml --api

# Diff local against active
fluxrig scenario diff main.yaml
```

**Flags for `import`**:
- `--api` - Sync with Mixer API immediately
- `--store-dir` - CAS store location (default: `~/.fluxrig/store`)

**Visualising a scenario**:

```bash
fluxrig scenario viz payment_flow.yaml
```

Generates a LikeC4 model from the scenario, for interactive drill-down and
topology validation. The output is plain text; view it with the `likec4` CLI.

### Operations & simulation examples

Use these patterns to drive a simulation:

```bash
# pull the latest compliance suite [Roadmap]
fluxrig scenario pull github.com/jaab-tech/compliance-tests

# run a specific certification scenario [Roadmap]
# This acts as a standalone runner, injecting traffic and asserting results.
fluxrig scenario run visa-cert:v2.1.0 --target https://my-rack:8583
```

### Telemetry queries

| Command | Description |
|---------|-------------|
| `fluxrig logs` | Query telemetry logs from Mixer (Remote) |
| `fluxrig metrics` | Query telemetry metrics from Mixer (Remote) |
| `fluxrig tail <node>` | Tail live logs from a specific node in real-time |
| `fluxrig inspect-logs` | Inspect binary WAL files on a Rack (Local) |
| `fluxrig configuration` | Show the runtime configuration for Mixers and Racks |
| `fluxrig check` | Verify connectivity to the bus, the Mixer API and local storage |

**Logs Flags**:

- `--api-url` - Mixer API URL (default: `http://localhost:8090`)
- `--limit` - Max records (default: 50)
- `--since` - Start time (e.g., `1h`)
- `--until` - End time (ISO timestamp)
- `--min-level` - Filter by level (TRACE, DEBUG, INFO, WARN, ERROR)
- `--entity` - Filter by entity name

**Metrics Flags**:

- `--api-url` - Mixer API URL (default: `http://localhost:8090`)
- `--limit` - Max records (default: 50)
- `--since` - Start time
- `--until` - End time
- `--name` - Filter by metric name
- `--entity` - Filter by entity name

### Observability query examples

Use these patterns to bridge the gap between business flows and system traces:

> `fluxrig trace <flux_id>`, to follow one business flow across every Rack, is
> **[Roadmap]**. Until it exists, a flow is followed by querying its `flux_id`
> through the commands below.

```bash
# View recent errors for a specific payment Gear
fluxrig logs --entity payment-processor --min-level error --since 5m

# Query a specific metric
fluxrig metrics --name flux.bus.messages_in
```

### Topology queries

| Command | Description |
|---------|-------------|
| `fluxrig topology status` | Show global synchronization status. |
| `fluxrig topology list` | List active deployments (racks and gears). |

---

## fluxrig-mixer (dynamic binary)
The orchestration server (requires CGO for DuckDB).

### `fluxrig-mixer`
Starts the Mixer server.

```bash
fluxrig-mixer --config fluxrig-mixer.toml --scenario scenario_01.yaml
```

**Flags**:

- `-c, --config` - Path to TOML configuration file (default: `fluxrig-mixer.toml` or `FLUXRIG_CONFIG`)
- `-s, --scenario` - Scenario reference to load on startup: file path (`./scenario.yaml`) or stored URN (`payment-flow:v1.0.0`). Empty to resume last active.
- `--auto-adopt` - Automatically approve and adopt any new Rack that connects. **For development use only.**

**Key Configuration Settings** (`[mixer]`):

- `api.port` - REST API port (default: `8090`)
- `mixer.data_dir` - Data directory (default: `./data`)
- `mixer.startup_scenario` - Alternative to `--scenario` flag (file path or `name:tag` URN)