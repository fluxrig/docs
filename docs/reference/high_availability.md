---
slug: /reference/platform/high-availability
title: High availability
---

# High availability

fluxrig runs in two modes. The embedded mode runs one Mixer with its own NATS server. The external mode runs several Mixers on a NATS cluster that the operator runs. Only the external mode keeps the control plane available when a Mixer stops.

In the embedded mode, the Mixer is a single point of the control plane. When it stops, enrollment, scenario deployment and the API wait until it returns. Racks keep processing messages with their active scenario meanwhile. They do not need the Mixer for the data plane.

In the external mode, several Mixers share one control plane. When one Mixer stops, the others continue to enroll Racks, accept control writes and serve the API. This page describes how the Mixers operate together and how to configure them. The configuration keys are in the [configuration reference](configuration.md).

## Two modes

| Mode | Setting | NATS | Mixers |
| :--- | :--- | :--- | :--- |
| Embedded (default) | `snake.external = false` | Inside the Mixer | One |
| External | `snake.external = true` | A [NATS](https://docs.nats.io) cluster that the operator runs | Several |

The embedded mode is one Mixer and its own NATS server. For high availability, run the external mode: the Mixers and the Racks connect to a NATS cluster that the operator runs, and the Mixers keep no NATS server of their own.

## The parts

The external mode has three parts.

1. **A NATS cluster.** It holds every stream and bucket: the messages, the events, the telemetry stream, the registry, the lease, the specs and the Wasm catalog. The operator runs it, and keeps its store and its encryption at rest.
2. **The write lease.** One Mixer holds the lease and does all control-plane writes (`registry.lease_enabled`, required in the external mode).
3. **Forwarding.** A Mixer that does not hold the lease sends each control write to the holder, and returns the answer of the holder to its caller.

All Mixers read the same registry. Thus each Mixer can answer a read, and each Mixer shows the same Racks and the same active scenario.

## Size

Use three NATS servers in one cluster. A replicated stream or bucket accepts a write only when a majority of its replicas confirms the write. With three replicas, the cluster continues when one server stops. With two, writes stop when one server stops.

Set `snake.replicas` (the streams) and `registry.replicas` (the registry, liveness, lease and spec buckets) to `3`. The two settings are separate. The Mixer warns when `snake.replicas` is below 3, and when `registry.replicas` is below `snake.replicas`.

The Mixers do not vote: the lease is a key in a NATS bucket. Two Mixers keep the control plane available when one stops.

## The NATS cluster

The Mixer checks the cluster when it starts. It refuses a NATS server older than 2.15.0, an account without JetStream, and `snake.replicas` above 1 on a server that is not in a cluster. A server that does not answer yet, and a stream or bucket that cannot be placed yet, are tried again for `snake.form_timeout` (default `"2m"`), so the Mixers can start while the cluster forms.

Set route pings on every server: `cluster { ping_interval: "10s", ping_max: 3 }`. A server that stops without closing its routes, for example a crash or a cut link, is found only when its pings stop. With the NATS defaults (a ping every 2 minutes) this took more than a minute in a test across a link with a round trip of approximately 220 ms, and the streams that the server led stopped during that time.

Set client pings on every server too, at the top level: `ping_interval: "10s"` and `ping_max: 3`. A client that stops answering keeps its subscriptions until its server drops it. Until then, the server sends that client its share of the Hello and heartbeat messages of the Racks, and each one comes again only after `snake.ack_wait`.

To give each Rack its own NATS user, see [Security reference](security_reference.md).

## The write lease

The lease is one key in the lease bucket (`registry.lease_bucket`). The key names the Mixer that holds the lease. The bucket deletes the key after `registry.lease_ttl`.

- The holder renews the key every `registry.lease_renew`.
- A Mixer that does not hold the lease tries to take it at the same interval. It gets the lease only when the key is not there.
- A holder that cannot renew within `registry.lease_ttl` stops writing. It uses its own clock for this, so it also stops when it cannot reach NATS.
- A Mixer that stops cleanly deletes the key. Another Mixer then takes the lease at its next attempt, and does not wait for `registry.lease_ttl`.
- When the holder stops without a clean stop, the key expires after `registry.lease_ttl`. Then another Mixer takes the lease.

The lease is not the only protection. Each registry write checks the revision of the record that it changes. A holder that pauses past its lease and then continues cannot overwrite a newer write.

`registry.lease_renew` must be at most half of `registry.lease_ttl`. The Mixer does not start otherwise. This rule gives the holder at least two renew attempts before the lease expires.

## Control writes

These writes go to the lease holder:

- Rack registration, approval, status change and removal.
- Scenario import and scenario activation.
- Spec import into the [shared spec store](spec_manager.md#shared-store).

A Mixer that does not hold the lease sends the write as a NATS request on `flux.mixer.control.<op>`. Each Mixer listens on these subjects, and only the holder answers. The sender waits `mixer.forward_timeout` for the answer. The caller gets the same result as from a local write, also the same error.

When no Mixer holds the lease, a control write fails. The API answers `503 Service Unavailable` for this failure. The API also answers `503` when the NATS cluster cannot apply a write yet, for example during the election of a stream leader. A client must send the request again after a short wait.

A Wasm import, and a change to the trusted vendor keys, are written by the Mixer that receives the request, to buckets that every Mixer reads. A vendor trusted or revoked through one Mixer counts on every Mixer at its next import.

## Racks

Set `snake.external = true` and `snake.url` on each Rack, with every NATS server of the cluster, comma-separated. The Rack connects to one server, and moves to another one in these conditions:

- The server stops, and the connection closes.
- The server stops answering, and the connection stays open. The Rack finds this after `snake.max_pings_out` pings with no answer, one each `snake.ping_interval`: 15 to 20 seconds with the defaults.
- The server is cut off from the cluster. The connection stays open, but no publish gets to a stream. After `snake.failover_after` such publishes in sequence, the Rack connects to a different server.

The Mixers share one durable queue group (`mixer.queue_group`) for the Hello and heartbeat messages of the Racks. One Mixer handles each message. When no Mixer can register a Rack yet, the Rack sends its Hello again until a Mixer answers.

A subscription reads through a consumer on one server of the cluster. When that server stops, the Rack creates a new consumer on a server that is still up. This occurs after `snake.consumer_missed_heartbeats` missed heartbeats, one each `snake.operation_timeout`. The new consumer starts at the first message that the Rack did not complete, so the Rack receives the messages that were sent in the interval.

A wire on the guaranteed lane delivers each message at least once. A message that its consumer sent through a server that stopped arrives again after `snake.ack_wait` (default `"10s"`). Messages can arrive out of order across the interval, and one message can arrive two times.

The registry keeps the presence of each Rack in the replicated liveness bucket, so each Mixer shows a Rack as `online`. The real-time statistics of a Rack (CPU and memory) stay in the memory of the Mixer that received its heartbeat. See [Identity & registry](registry.md#heartbeats-and-presence).

The Mixers and the Racks must run the same version.

## The active scenario

The Mixer that receives an activation request sends the change of the active-scenario pointer to the lease holder. When the holder confirms the write, the Mixer sends the scenario to the Racks. Each other Mixer reads the pointer every `mixer.follow_interval`. When the pointer changes, it loads the new scenario. Thus each Mixer reports the same active scenario, and each Mixer can send it to a Rack that enrolls.

## Telemetry

`observability.storage.type` sets where the Mixers keep the telemetry of the Racks.

Shared telemetry stores contain Rack logs, metrics and spans. Apply the storage safeguards in the [configuration reference](configuration.md).

| Type | Where | With several Mixers |
| :--- | :--- | :--- |
| `local` (default) | The DuckDB of each Mixer | Each Mixer keeps its own copy, and misses the telemetry that arrived while it was stopped. The Mixer warns about this |
| `s3` | Parquet files in an S3-compatible bucket | One copy. Every Mixer answers telemetry queries up to the last batch |
| `nats` | Parquet files in a NATS object store, with `snake.replicas` copies | One copy, with no other service to run |
| `otlp` | The collector of the operator, through OTLP over HTTP | The Mixers keep no telemetry, and the telemetry queries of the API answer `501`. One batch can arrive two times when the holder stops after the export and before the confirmation |

The three shared types need the lease. Only the holder reads the telemetry stream, and it confirms each batch only after the store has it. A new holder continues from the last confirmed batch. With `s3` and `nats`, a batch that is written again replaces its own files, so the store keeps each record one time. Every Mixer keeps the live metrics of each Rack. The keys are in the [configuration reference](configuration.md).

## Monitor the lease

Each Mixer shows the lease in three places:

- **`GET /api/v1/health`**. The answer has a `lease` object: `holder` (the `machine_id` of the Mixer that holds the lease), `holder_since`, `renewed` (the last renewal of the holder), and `held` (true on the Mixer that holds it). When no Mixer holds the lease, `holder` is not in the answer. When the Mixer cannot read the lease, `error` is `"unavailable"`; it does not expose the raw cause. The status of the answer is still `200`. A Mixer with `registry.lease_enabled = false` has no `lease` object.
- **Metrics.** `flux.mixer.lease.held` is `1` on the Mixer that holds the lease and `0` on the others. When the Mixers operate correctly, the sum is `1`. `flux.mixer.lease.transitions` counts the changes of this Mixer, with the attribute `event`: `acquired`, `lost` or `released`. Read them with `fluxrig metrics` or `GET /api/v1/telemetry/metrics`.
- **The log.** The Mixer writes a line when it acquires, loses or releases the lease.

## Failures

**A Mixer stops.** Another Mixer takes the lease (see "The write lease"). Until then, a control write answers `503`. Reads and the data plane continue. To replace the Mixer, start a new one with the same `cluster.key`, the same configuration and an empty `store.dir`.

**A Mixer is cut off from NATS.** It refuses control writes with `503`, and operates again when the connection comes back.

**One NATS server stops.** The streams that it led elect a new leader, and control writes come back after the election. The Racks and the Mixers that were connected to it move to another server. Start the server again with its store: it gets the writes it missed from the other servers.

**Two NATS servers stop.** The cluster has no majority: the streams stop, and with them the control writes and the guaranteed lane. The Racks continue to run their gears. A Rack keeps in its outbox the messages that it cannot publish, up to `rack.outbox_max_messages`, and sends them when the bus comes back. The outbox is in memory only: a Rack that restarts during this time loses it.

**A Rack is online and runs no gears.** The Rack failed to apply its scenario five times in sequence, and the scenario went to the dead letter stream, on `flux.dlq.flux.rack.<rack>.scenario`. The log of the Rack gives the reason. Correct the cause, then activate the scenario again: an activation makes every Rack of the scenario apply it again, also a Rack that already has it.

## The cluster key

All Mixers must use the same `cluster.key`. The cluster key signs the Rack passports, and a Rack refuses a passport that a different key signed. With `registry.lease_enabled` set, a Mixer does not start when its `cluster.key` is different from the key that the registry records.

Copy `cluster.key` from the first Mixer to the other Mixers before they start for the first time. Keep a copy with the backup of the NATS store. See [Operating fluxrig](operations.md) for how to back up this key.

## Lab results

Five full runs passed every scenario in the lab. Control writes came back 30 to 33 s after the kill of the lease holder. They came back 2 to 11 s after the kill of the NATS server that led the message stream. No acknowledged control write was lost.

These numbers are observations, not guarantees. For the testbed, the conditions and the full results, follow [Verify failover in a lab](../tutorials/ha_failover_lab.md).

## Configuration example

This example is from a test cluster of three NATS servers, three Mixers and two Racks, with TLS and a NATS user for each client certificate. The addresses of the test are replaced with names.

One NATS server. Change `server_name` and the certificate for each server:

```text
server_name: n1
port: 4222
ping_interval: "10s"
ping_max: 3
jetstream { store_dir: "/data/js" }
cluster { name: fluxrig-ha, port: 6222, ping_interval: "10s", ping_max: 3, routes: [nats://n1:6222, nats://n2:6222, nats://n3:6222] }
tls { cert_file: "/etc/fluxrig/tls/n1.crt", key_file: "/etc/fluxrig/tls/n1.key", ca_file: "/etc/fluxrig/tls/ca.crt", verify_and_map: true }
authorization { users: [
  { user: "CN=mixer-m1,O=fluxrig" }
  { user: "CN=mixer-m2,O=fluxrig" }
  { user: "CN=mixer-m3,O=fluxrig" }
] }
```

The users of the Racks come from `fluxrig scenario permissions` and go in the same `users` list.

One Mixer. Change `base.name` and the certificate for each Mixer:

```toml
[base]
name = "mixer-m1"

[mixer]
queue_group = "fluxrig-mixers"
follow_interval = "5s"
forward_timeout = "5s"

[snake]
external = true
url = "tls://n1:4222,tls://n2:4222,tls://n3:4222"
replicas = 3
form_timeout = "2m"
root_ca_file = "/etc/fluxrig/tls/ca.crt"
client_cert_file = "/etc/fluxrig/tls/mixer-m1.crt"
client_key_file = "/etc/fluxrig/tls/mixer-m1.key"

[store]
dir = "/data"
cluster_key_file = "cluster.key"

[registry]
lease_enabled = true
lease_ttl = "30s"
lease_renew = "10s"
replicas = 3

[observability.storage]
type = "nats"
```

One Rack:

```toml
[base]
name = "rack-r1"

[snake]
external = true
url = "tls://n1:4222,tls://n2:4222,tls://n3:4222"
root_ca_file = "/etc/fluxrig/tls/ca.crt"
client_cert_file = "/etc/fluxrig/tls/rack-r1.crt"
client_key_file = "/etc/fluxrig/tls/rack-r1.key"
```
