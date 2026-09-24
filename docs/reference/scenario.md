---
slug: /reference/platform/scenarios
title: Orchestration scenarios
---

# Orchestration scenarios

A **Scenario** is a declarative YAML file that defines the operational topology for the entire environment, which Gears run, how they connect, and what specs they reference.

## Lifecycle

1. **Author** a scenario YAML (locally or in version control).
2. **Import** into the [content-addressable store](spec_manager.md#cas) with a `name:tag` reference:

   ```bash
   fluxrig scenario import payment_flow.yaml --name payment-flow --tag v1.0.0
   ```

3. **Load** at Mixer startup via CLI flag or configuration:

   ```bash
   fluxrig-mixer -s payment-flow:v1.0.0     # From store
   fluxrig-mixer -s ./scenarios/test.yaml    # From file
   fluxrig-mixer                             # Resume last active
   ```

4. **Activate**, the Mixer validates, persists, and wires the scenario.

### Hot-reloading & rollbacks

> [!IMPORTANT]
> **Implementation Note**: Currently, applying a scenario update triggers a **full Gear Restart cycle**. All active gears are stopped and re-initialized with the new configuration. This introduces a sub-second processing gap and closes established I/O connections (e.g., TCP sessions). 

**Future Roadmap**: Achieving true **Zero-Downtime Reload** (where unaffected gears continue processing while new configurations are swapped in) is a primary engineering objective for the **future milestone**.

If you import a broken scenario containing invalid wires or missing specs, the Mixer performs **pre-flight validation** and rejects the transition, ensuring the cluster remains on the last known good state.

## Startup resolution

When the Mixer starts, the `startup_scenario` reference is resolved using three strategies:

| Reference Format | Example | Behavior |
| :--- | :--- | :--- |
| *(empty)* | `""` | Resume the last active scenario from the store (non-fatal if none found). |
| File path (contains `/`) | `./scenario.yaml` | Read YAML from file, import, and activate. Fatal on error. |
| `name:tag` URN | `payment-flow:v1.0.0` | Resolve from the CAS store via `manager.Load()`, import, and activate. |

The `file://` prefix is optional for file paths (e.g., `file://./scenario.yaml`).

## YAML schema

```yaml
meta:
  name: Payment Processing       # Required. Human-readable name.
  version: 1.0.0                 # Required. Semantic version.

racks:
  - name: rack-prod-01           # Explicit target.
    config:
      location: "montevideo-dc1"

  - group: edge-gateways         # Target group.
    match:
      labels:
        role: "gateway"
    defaults:
      log_level: "info"

gears:
  - name: iso-inbound
    type: io_iso8583
    deploy: edge-gateways        # References a group or specific rack.
    config:
      mode: "server"
      bind: ":8583"

wires:
  - from: iso-inbound.out
    to:   router.in
```

> [!NOTE]
> **Wires are unidirectional.** Each entry moves messages one way: from an **output** port (`from`) to an **input** port (`to`). External TCP sockets are bidirectional, but that duality ends at the I/O gear boundary: a request/response exchange always requires a **pair** of wires, one carrying requests away from the I/O gear's `out` port and one delivering responses back to its `in` port. See [the port model](../architecture/gear.md#the-port-model).

## Labels the diagram reads

`fluxrig scenario viz` derives a topology from gears and wires. Two things it
cannot derive are who is on the far side of a socket, and what a Rack is for. A
scenario can say both, and nothing else reads these labels.

```yaml
racks:
  - name: rack-fluxrig
    labels:
      role: "the switch under test"      # becomes the Rack's description

gears:
  - name: scheme-in
    type: io_iso8583
    labels:
      peer: "Card scheme"                # names the far side of the socket
      peer_played_by: "iso8583-tool"     # who stands in for it, if anyone
    config:
      mode: "server"
      bind: ":8583"
```

Without `peer`, an I/O gear's counterparty is drawn as `External clients
(:8583)`, which describes a socket rather than who is on it. `peer_played_by` is
for test topologies: a reader deciding whether a suite proves anything needs to
know which participants are real and which are simulated, and that belongs on the
box rather than in prose elsewhere.

### Rendering the diagram

`fluxrig scenario viz <scenario>.yaml -o <dir>` writes a self-contained
[LikeC4](https://likec4.dev) model (specification, model and views) into `<dir>`.
The view ids are stable: `index` for the whole scenario, `of_rack_<name>` for one
Rack expanded, `gear_<name>` for a gear's neighbourhood. Point LikeC4 at the
directory to render or export it. Because the model is generated from the same
YAML the Racks run, the diagram and the wiring cannot drift apart.

Neither is needed when both ends are in the scenario. A client whose `connect`
matches a server's `bind` is drawn as one relationship between those two gears,
with no external box on either side.

## Gear deployment

The `gears` section defines which logic units are active and where they are executed.

```yaml
gears:
  - name: iso-inbound
    type: io_iso8583
    deploy: rack-alpha           # Optional. Target Rack name.
```

### Global gears

If a gear does not specify a `deploy` target, it is treated as a **Global Gear**. 

*   **Behavior**: The gear will be pushed to and executed by **every Rack** that connects to the Mixer and receives the scenario.
*   **Use Case**: This is ideal for "Zero-Config" Getting Started scenarios or for deploying universal monitoring/diagnostic gears across a distributed cluster without knowing the dynamic Rack names in advance.

A gear naming a `deploy` target is validated at activation: the target must
exist as an active Rack in the registry, or activation fails naming it. Import
stays permissive, so a scenario can be filed before its Racks enroll. A scenario
with no push targets activates into nothing, which the Mixer logs as a warning.

## Pipe configuration

The `wires` (or `pipes`) section defines how data flows between Gears.

> [!NOTE]
> **Endpoint grammar.** A wire endpoint is `gear.port` (the rack is taken from the gear's `deploy`) or `rack.gear.port` (an explicit rack / replica instance). Every segment is dot-free (**port names use underscores for roles**, as in `in_reply` and `out_scheme_a`, never dots), so `a.b.c` is always `rack.gear.port`. A wire naming an undefined rack/gear, or a port a gear does not declare, is rejected at import. See [the port model](../architecture/gear.md#wire-endpoint-naming).

### Lanes

Each wire travels on one of two lanes. The optional `lane` field of a wire chooses it.

| `lane` | Where the messages go | Delivery | Needs the Mixer |
| :--- | :--- | :--- | :--- |
| `hot` | Through the memory of the Rack, from the gear that emits to the gear that consumes. Nothing is stored and nothing is sent to the Mixer. | At most once, in order. A message still queued when the Rack process ends is lost. | No |
| `guaranteed` | Over the bus, the NATS JetStream server embedded in the Mixer, which stores each message before the emitting gear is told it was accepted. | Stored, encrypted at rest by default (see [data at rest](../architecture/security.md#data-at-rest-and-in-logs)). | Yes |
| not set | `hot` when both gears run on the same Rack, `guaranteed` when they run on different Racks. | | |

A wire between gears on different Racks is always on the guaranteed lane. A wire that asks for `hot` between gears on different Racks is rejected at import.

What the hot lane changes for an operator:

*   **Nothing rests on a disk.** A message on a hot wire, a card number included, is in the memory of the Rack and nowhere else. A wire that asks for `guaranteed` inside one Rack is stored on the Mixer.
*   **It keeps running while the Mixer is away.** A hot wire needs no bus, so a flow that stays inside one Rack keeps processing when the Mixer is unreachable. See [A Rack without the Mixer](operations.md#a-rack-without-the-mixer).
*   **A slow consumer slows the emitter.** Each hot wire holds `rack.lane_queue_size` messages (1024 by default). A gear that emits into a full queue waits `rack.lane_send_timeout` (5 seconds by default) and then gets an error, as it would from a bus that does not answer. Gears that hand messages to each other in a cycle can fill each other's queues, and the timeout is what turns that into an error and not a stall.
*   **A stop that is asked to be graceful delivers what is queued**, within `rack.drain_timeout`. A crash does not.

### Example: choosing a lane

```yaml
wires:
  # Inside the Rack: through memory. This is the default, and `lane: "hot"` says the same.
  - from: "gateway.out"
    to: "gateway.in"

  # Storage is asked for even though both gears are on one Rack.
  - from: "gateway.out"
    to: "audit.in"
    lane: "guaranteed"
```

## Versioning & storage

Scenarios are stored in the same Content-Addressable Store (CAS) as Specs. Each imported scenario is:

- Hashed (SHA-256) and stored as an immutable blob.
- Indexed under `name → tag → hash`.
- Retrievable via `name:tag` or `sha256:hash`.

The special tag `latest` resolves to the highest SemVer tag for a given name.

See [Spec & Scenario Manager](spec_manager.md) for details on CAS internals and the `fluxrig scenario` CLI.