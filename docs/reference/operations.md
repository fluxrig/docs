---
slug: /reference/operations/running
title: Operating fluxrig
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Operating fluxrig

How the two processes are started, how a Rack is brought into a fleet, what
changing behaviour looks like, what normal looks like, and what the system does
on its own before anyone looks.

## What you are running

Two processes, with different jobs and different failure consequences.

| | Role | If it stops |
| :--- | :--- | :--- |
| **Rack** (`fluxrig`) | Runs a scenario: receives, transforms and forwards traffic | Traffic on that node stops |
| **Mixer** (`fluxrig-mixer`) | Enrolls Racks, deploys scenarios, collects telemetry | Traffic keeps flowing; nothing new can be deployed |

A Rack connects outbound to the Mixer over mTLS. The Mixer never dials a Rack,
so a Rack behind NAT or inside an isolated segment needs no inbound rule.

## Starting the Mixer

```bash
fluxrig-mixer -c fluxrig-mixer.toml            # config file
fluxrig-mixer -c fluxrig-mixer.toml -s flow.yaml   # and a scenario to load at boot
```

It listens on two ports: the API on **8090** and the bus on **4222**. The bus is
embedded, not a dependency: the Mixer runs its own NATS server (the Snake), so
there is no broker to deploy beside it.

That is a constraint as well as a convenience. **The bus cannot be replaced by an
existing NATS deployment**: the Mixer starts its own on every boot, and there is
no configuration that points it at another. What is configurable is where it
listens (`snake.port`) and what address Racks are given for it (`snake.url`), so
the bus can move host and port but remains the Mixer's.

On first start it writes its own identity and creates what it needs. Under the
store directory (`./data` by default):

| File | What it is |
| :--- | :--- |
| `cluster.key` | The Ed25519 authority that signs Rack passports and Wasm payloads |
| `mixer.flux` | The Mixer's own passport, signed by that key |
| `flux.duckdb` | The registry and telemetry |

**`cluster.key` is the one to back up.** The Mixer generates it when it is
missing, and then fails to verify the `mixer.flux` beside it, so a Mixer that
loses its key and keeps its data refuses to start. Every passport it ever issued
was signed by that key.

## Bringing a Rack into the fleet

A Rack enrolls itself on first start. It sends a hello, the Mixer records it as
`pending`, and it stays there until somebody says yes:

```bash
fluxrig admin racks list           # who is asking, and who is already in
fluxrig admin racks approve <id>   # let one in
```

Approval is a deliberate step: an enrolled Rack receives scenarios, so adopting
one is granting it work. A Mixer can be told to adopt automatically with
`--auto-adopt`, which is for a development machine and says so.

Once approved, the Rack holds a signed passport (`rack.flux` in its data
directory) and its own identity. Suspended Racks come back with
`fluxrig admin racks activate <id>`; `fluxrig admin racks remove <id>` takes one
out of the registry for good.

## Changing what runs

Behaviour is a scenario, and a scenario is a versioned artefact rather than a
file someone edits in place. The change is filed, then deployed.

```bash
fluxrig scenario diff payment_flow.yaml          # against what is running now
fluxrig scenario import payment_flow.yaml \
  --name payment-flow --tag v1.2.0 --dry-run     # validate, change nothing
fluxrig scenario import payment_flow.yaml \
  --name payment-flow --tag v1.2.0 --api         # file it and send it to the Mixer
fluxrig topology list                            # what is deployed where
```

A protocol spec travels the same way, and the two are related: a scenario names
the specs it needs, and deploying it carries them.

```bash
fluxrig spec import iso8583-v87-ascii.yaml
fluxrig spec list                                # what the store holds
fluxrig spec history iso8583-v87-ascii           # every version of one spec
```

Both live in the [content-addressable store](spec_manager.md#cas), so a version
is immutable: the same name and tag always resolve to the same bytes. A URN is
resolved when a gear starts, not per message, so importing a newer version
changes nothing until the scenario restarts.

## What normal looks like

```bash
fluxrig topology status            # what the Mixer believes about the fleet
fluxrig metrics                    # recent telemetry
curl -s localhost:8090/api/v1/health
```

Two numbers are worth watching per gear, because together they say whether work
is going in and coming out: `flux.gear.messages_in` and
`flux.gear.messages_out`. A gear whose `messages_in` climbs while
`messages_out` does not is dropping or erroring, and that comparison is faster
than reading any log.

Beside them, `flux.gear.errors` and `flux.gear.processing_time_ms` on every
gear. I/O gears add `flux.port.bytes_in`, `flux.port.bytes_out`,
`flux.port.messages_in`, `flux.port.messages_out`,
`flux.port.connections_active` and `flux.port.connections_total`. The bus reports `flux.bus.publish_count` and
`flux.bus.publish_errors`. The ISO 8583 codec adds
`flux.codec.iso8583.fields_count` and, when validation is on,
`flux.iso8583.violations`.

Telemetry lands in DuckDB on the Mixer and is flushed to Parquet. See
[Telemetry and analytics](telemetry_analytics.md).

## A Rack keeps running without the Mixer

This is a design decision, not a fallback. A Rack that has enrolled resumes its
last scenario from local state, and if the bus is unreachable at startup it logs
`Starting in OFFLINE Mode` and carries on processing.

The consequence for an operator: **a silent Mixer does not stop money moving**,
and a Rack that looks absent from `topology status` may be serving traffic
normally. Check the Rack before declaring an outage.

A Rack that has never enrolled and cannot reach the bus refuses to start,
because it has no identity and no scenario to resume.

## What degrades, and what fails

| Situation | What happens |
| :--- | :--- |
| Mixer unreachable | Racks keep processing. Telemetry queues locally; no scenario changes land. |
| A gear returns an error | Follows that gear's `on_error`: `reject` (out the error port), `drop` (discard), `kill` (fail the gear). |
| A destination stops answering | The Conductor's ticket expires and surfaces on the error port. See [Conductor](gears/conductor.md). |
| A message breaks a spec rule | Depends on the codec's `validation`: `off`, `warn` (logged and counted) or `enforce` (fails the message, then `on_error`). |
| An enrichment call exceeds its deadline | The pipeline continues with the fact absent, rather than holding the transaction. See the [roaming tutorial](../tutorials/roaming_enrichment.md). |

## When something is wrong

**Decide which half first.** A Rack and a Mixer fail independently, and the
answer changes what you look at next.

```bash
fluxrig check                      # bus, Mixer API and local storage, from this node
curl -s localhost:8090/api/v1/health
```

If `check` passes and traffic is still wrong, the fault is in the scenario or a
gear, not in the plumbing.

A Mixer that will not start is usually saying so: it verifies `mixer.flux`
against `cluster.key` before anything else, and refuses rather than issuing
identities it cannot stand behind.

**Logs.** `fluxrig logs` queries the Mixer for a fleet view; `fluxrig tail
<node>` follows one Rack live; `fluxrig inspect-logs` reads the binary WAL on the
Rack itself, which is the only one that works when the Mixer does not.

**Which spec ran.** Every message the ISO 8583 codec handles carries
`codec.spec_hash`, `codec.spec_id` and `codec.spec_version`. When two Racks
behave differently on what looks like the same traffic, compare the hashes: they
are running the same spec or they are not, and nothing else answers that
question.

## Configuration

Ports, timeouts, store locations and TLS are in the
[configuration reference](configuration.md). Two rules hold throughout: every
wait has a timeout, and every timeout is a configuration field with a documented
default.
