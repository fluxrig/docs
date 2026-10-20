---
slug: /reference/operations/running
title: Operating fluxrig
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Operating fluxrig

This page explains how you start the two processes. It explains how you bring a Rack into a fleet. It describes what a behaviour change looks like. It describes what normal operation looks like. It describes what the system does on its own before anyone looks.

## What you are running

Two processes run the system. They have different jobs and different failure consequences.

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

That is a constraint as well as a convenience. **No existing NATS deployment can replace the bus**: the Mixer starts its own on every boot, and no configuration points it at another. You can configure where it listens (`snake.port`) and what address Racks receive for it (`snake.url`). The bus can move host and port but remains the Mixer's.

On first start it writes its own identity and creates what it needs. It places these files under the
store directory (`./data` by default):

| File | What it is |
| :--- | :--- |
| `cluster.key` | The Ed25519 authority that signs Rack passports and Wasm payloads |
| `mixer.flux` | The Mixer's own passport, signed by that key |
| `snake/` | The embedded NATS store: messages, and the registry (Rack records, scenario files, the active scenario) |
| `flux.duckdb` | Telemetry, and the topology rows derived from the active scenario |

**`cluster.key` is the one to back up.** The Mixer generates it when it is
missing, and then fails to verify the `mixer.flux` beside it, so a Mixer that
loses its key and keeps its data refuses to start. That key signed every passport it ever issued.

The key that encrypts the message store is derived from `cluster.key`. So the
key and the data belong together: a store restored without its key does not
open, and a key restored without its store is a new authority that orphans
every Rack holding a passport from the old one. Back them up as one unit and
restore them as one unit.

### Backing up and restoring

Stop the Mixer, copy the whole store directory, start it again:

```bash
kill -TERM "$(pidof fluxrig-mixer)"   # it drains and exits
cp -a ./data /backup/mixer-$(date +%Y%m%d)
fluxrig-mixer -c fluxrig-mixer.toml
```

Copying a live `flux.duckdb` is not safe: its write-ahead log is a separate
file, and a copy that takes one without the other is a corrupt pair. For zero
outage, snapshot the volume `store.dir` lives on and copy the snapshot out.

To restore, stop the Mixer, move the current store directory aside without
deleting it, copy the backup into place, and start. A clean start needs nothing
else. If the Mixer instead reports that the store did not open with the current
key, the key and the store disagree: restore the key that belongs to that
store. Losing the key entirely means choosing between re-rooting the authority
(`snake.store_old_key_file`, if the old store key was saved with
`fluxrig keys store-key`) and starting clean with a new registry.

Back up the whole directory, not only the key: it also holds the message and
key-value store with the registry (`snake/`), the analytics database
(`flux.duckdb`), and the specs (`store/`). The registry keeps only a hash of
each Rack's secret. The analytics database and the Parquet files are not
encrypted, so treat a backup as sensitive rather than as ciphertext.

## Bringing a Rack into the fleet

A Rack enrolls itself on first start. It sends a hello. The Mixer records it as
`pending`. It stays there until somebody says yes:

```bash
fluxrig admin racks list           # who is asking, and who is already in
fluxrig admin racks approve <id>   # let one in
```

Approval is a deliberate step: an enrolled Rack receives scenarios, so adopting
one grants it work. You can tell a Mixer to adopt automatically with
`--auto-adopt`, which is for a development machine and says so.

Once the Mixer approves it, the Rack holds a signed passport (`rack.flux` in its data
directory) and its own identity. Suspended Racks come back with
`fluxrig admin racks activate <id>`. `fluxrig admin racks remove <id>` takes one
out of the registry for good.

## Changing what runs

Behaviour is a scenario. A scenario is a versioned artefact rather than a
file someone edits in place. You file the change. Then you deploy it.

```bash
fluxrig scenario diff payment_flow.yaml          # against what is running now
fluxrig scenario import payment_flow.yaml \
  --name payment-flow --tag v1.2.0 --dry-run     # validate, change nothing
fluxrig scenario import payment_flow.yaml \
  --name payment-flow --tag v1.2.0 --api         # file it and send it to the Mixer
fluxrig topology list                            # what is deployed where
```

A protocol spec travels the same way. The two relate to each other. A scenario names
the specs it needs. The Mixer carries them when it deploys the scenario.

```bash
fluxrig spec import iso8583-v87-ascii.yaml
fluxrig spec list                                # what the store holds
fluxrig spec history iso8583-v87-ascii           # every version of one spec
```

Both live in the [content-addressable store](spec_manager.md#cas), so a version
is immutable: the same name and tag always resolve to the same bytes. A gear resolves a URN when it starts, not per message, so you change nothing by importing a newer version until the scenario restarts.

## What normal looks like

```bash
fluxrig topology status            # what the Mixer believes about the fleet
fluxrig metrics                    # recent telemetry
curl -s localhost:8090/api/v1/health
```

Two numbers are worth watching per gear, because together they say whether work
is going in and coming out: `flux.gear.messages_in` and
`flux.gear.messages_out`. A gear whose `messages_in` climbs while
`messages_out` does not drops or errors, and that comparison is faster
than reading any log.

Beside them, watch `flux.gear.errors` and `flux.gear.processing_time_ms` on every
gear. I/O gears add `flux.port.bytes_in`, `flux.port.bytes_out`,
`flux.port.messages_in`, `flux.port.messages_out`,
`flux.port.connections_active` and `flux.port.connections_total`. The bus reports `flux.bus.publish_count` and
`flux.bus.publish_errors`. The ISO 8583 codec adds
`flux.codec.iso8583.fields_count` and, when validation is on,
`flux.iso8583.violations`.

Telemetry lands in DuckDB on the Mixer, and the Mixer flushes it to Parquet. See
[Telemetry and analytics](telemetry_analytics.md).

## A Rack without the Mixer

A Rack keeps three things in local state: its identity (the Passport), the last scenario it applied, and a write-ahead log of its logs. What it does when the Mixer is unreachable depends on when it lost it.

| When | What the Rack does |
| :--- | :--- |
| At start, with a Passport | It waits `snake.offline_start_timeout` for the bus (3 seconds by default) and logs `Starting in OFFLINE Mode`. It then starts the last scenario it applied, from its local copy, if none of that scenario's wires uses the bus (see [lanes](scenario.md#lanes)). A scenario that does needs the bus, and waits for it as a whole: the Rack says so in its log and never runs part of it. |
| At start, without a Passport | It refuses to start: it has no identity and no scenario to resume. |
| While running | The process stays up and reconnects by itself when the Mixer returns. A wire between two gears of the same Rack keeps working, because it runs through the Rack's memory. A bus-bound emission made while the bus is down is parked in a bounded in-memory outbox and forwarded when the bus returns, instead of failing. What needs the bus stops only if it was never emitted: the heartbeat, and the export of metrics and traces. |

A Rack that started without the Mixer probes the bus every `snake.offline_retry_interval` (5 seconds by default). When the bus answers, the Rack joins the Mixer. It says hello, receives its Passport, starts its telemetry, and listens for scenarios. It does all this **without stopping the gears it runs**. A client that connected to one of them stays connected, and the Rack does not start the gears again. If the Mixer then sends a scenario identical to the one running, nothing restarts. The Rack applies a different scenario as it applies any new scenario. It restarts the gears the scenario changes. Parked emissions drain in the same step.

The outbox holds `rack.outbox_max_messages` (10000) emissions. When full the oldest is dropped and counted (`flux.outbox.dropped`). It is memory only: a Rack restart with a non-empty outbox loses it.

The drain keeps the order `rack.outbox_order` sets. With `subject`, the default, an emission never reaches the bus ahead of an older one on the same subject. A reversal parked after its authorization arrives after it. While a subject still has parked emissions, a new emission on it queues behind them. With `none`, the drain forwards whatever the bus takes, in the order the bus takes it.

The drain drops a parked emission only when the bus reports that it can never accept it: the emission fails validation, cannot be encoded, or exceeds the payload limit. Any other refusal keeps it parked. The drain tries a refused emission `rack.outbox_drain_retries` (3) more times, then moves on. With `subject` order, the rest of that subject waits for the next pass. A pass that forwards nothing stops the drain with everything still parked. A drain starts when the Rack rejoins the Mixer, and when the bus accepts an emission again.

The bus is the NATS server embedded in the Mixer. A wire between two gears on one Rack does not use it, unless the scenario asks for the guaranteed lane. The consequence for an operator is this. **A silent Mixer stops what crosses between Racks and what uses the guaranteed lane. It does not stop a flow that stays inside one Rack.** A Rack that looks absent from `topology status` may be serving such a flow normally. Verify the Rack before declaring an outage. Restore the Mixer for everything that crosses between Racks.

While there is no bus, the gears of a Rack still signal each other in memory (a link that goes down, a connection to close). Commands from the Mixer, such as the simulator's, reach the gears once the Rack has joined it.

The Rack writes logs to a local write-ahead log and ships them when the bus is reachable again, up to `store.wal_max_size_mb` (500 by default). It does not keep metrics and traces: their export fails while the Mixer is unreachable.

### What is not there yet `[Roadmap]`

*   **A guaranteed lane local to the Rack.** A NATS leaf node in each Rack would keep on-disk storage, with retention limits. It would serve the wires that ask for it while the Mixer is away.

## What degrades, and what fails

| Situation | What happens |
| :--- | :--- |
| Mixer unreachable | Racks stay up and reconnect by themselves. Flows that stay inside one Rack keep running; bus-bound emissions are parked and forwarded when the bus returns. Logs are kept in a local write-ahead log, and no scenario changes land. See [A Rack without the Mixer](#a-rack-without-the-mixer). |
| A gear returns an error | Follows that gear's `on_error`: `reject` (out the error port), `drop` (discard), `kill` (fail the gear). |
| A destination stops answering | The Conductor's ticket expires and surfaces on the error port. See [Conductor](gears/conductor.md). |
| A message breaks a spec rule | Depends on the codec's `validation`: `off`, `warn` (logged and counted) or `enforce` (fails the message, then `on_error`). |
| An enrichment call exceeds its deadline | The pipeline continues with the fact absent, rather than holding the transaction. See the [roaming tutorial](../tutorials/roaming_enrichment.md). |

## When something is wrong

**Decide which half first.** A Rack and a Mixer fail independently. The
answer changes what you look at next.

```bash
fluxrig check                      # bus, Mixer API and local storage, from this node
curl -s localhost:8090/api/v1/health
```

If `check` passes and traffic is still wrong, the fault is in the scenario or a
gear, not in the plumbing.

A Mixer that will not start usually says so: it verifies `mixer.flux`
against `cluster.key` before anything else, and refuses rather than issue
identities it cannot stand behind.

A Mixer also refuses to start when its management API cannot start: when
`api.auth_token` is not set, or the API port is in use. It exits with code 1
and names the cause. A Mixer without its API cannot be managed, and nothing
can upload a spec to it.

**A Rack that lost its identity.** A Rack keeps its identity in its
`state_dir`. When that directory is lost or wiped, the Rack starts again with
no identity and sends a Hello with its name. The Mixer refuses the Hello
because the name already belongs to the old record, and logs `Registration
denied: name conflict`. The Rack logs `Enrollment Timeout. No Passport
received.` The refusal is deliberate: a name is not proof of identity, so a
new process cannot take over the record of a Rack by using its name.

To let the Rack enroll again, remove the old record:

1. Stop the Rack.
2. Find the record of the name:

   ```bash
   fluxrig admin racks list
   ```

3. Remove it. `--force` skips the confirmation prompt:

   ```bash
   fluxrig admin racks remove <id>
   ```

4. Start the Rack. It enrolls as a new Rack, with a new identity.

A Mixer that fails while provisioning a stream names the stream and the
directory. Do not delete anything yet: stop the Mixer and diagnose the
streams first.

```bash
fluxrig-mixer -c fluxrig-mixer.toml --repair
```

The mode starts the embedded server without provisioning, checks each
stream it manages, and reports: healthy, drifted (config differs),
absent, or unreadable. It fixes drift by updating the stream config and
creates what is absent; both lose nothing. An unreadable stream is only
recreated with `--force`, which deletes it and reports exactly what was
discarded. Run it again afterwards: a silent run with exit 0 means every
stream is healthy.

**Logs.** `fluxrig logs` queries the Mixer for a fleet view. `fluxrig tail
<node>` follows one Rack live. `fluxrig inspect-logs` reads the binary WAL on the
Rack itself, which is the only one that works when the Mixer does not.

**Which spec ran.** Every message the ISO 8583 codec handles carries
`codec.spec_hash`, `codec.spec_id` and `codec.spec_version`. When two Racks
behave differently on what looks like the same traffic, compare the hashes. They
run the same spec or they do not, and nothing else answers that
question.

## Configuration

The [configuration reference](configuration.md) lists ports, timeouts, store locations and TLS. Two rules hold throughout: every
wait has a timeout, and every timeout is a configuration field with a documented
default.
