---
slug: /tutorials/quickstart
title: 5-Minute Quickstart
---

# 5-Minute Quickstart

This guide will walk you through a fully functional local deployment of **fluxrig** in under 5 minutes. You will spin up the control plane, enroll an edge node, and start processing telemetry data without writing any custom code or configuring complex registries.

> [!TIP]
> **Zero Config First-Run**: By default, fluxrig automatically generates necessary cryptographic keys and embedded database files in your working directory. You don't need a pre-existing complex database or secret manager setup to get started!

## Prerequisites

*   **Linux or macOS**
*   **Go 1.26+** (if compiling from source)
*   **Make**

---

## 1. Get the binaries

Currently, building from source is the only available option. Clone the repository and run `make build`:

```bash
git clone https://github.com/jaab-tech/fluxrig.git
cd fluxrig
make build
```

This will produce the two core binaries in the `./bin` directory:
*   `fluxrig-mixer`: The centralized Control Plane (Registry, Telemetry, NATS Bus).
*   `fluxrig`: The unified CLI and Edge Node (The Rack).

---

## 2. Start the Mixer (Control Plane)

The Mixer acts as the brain of your rig. It manages edge node enrollment, maintains the system topology, and ingests telemetry.

For this quickstart, we will use the built-in `getting_started.yaml` scenario and the `--auto-adopt` flag. The `--auto-adopt` flag bypasses the manual approval step for new edge nodes, making local development seamless.

```bash
# In your first terminal window:
./bin/fluxrig-mixer --auto-adopt examples/scenarios/getting_started.yaml
```

**What just happened?**
1. The Mixer initialized an embedded **DuckDB** telemetry store.
2. It started an embedded **NATS JetStream** server on port `4222`.
3. It imported the `getting_started.yaml` scenario and set it to active.

---

## 3. Start a Rack (Edge Node)

The Rack is the edge execution node. It connects to the Mixer, receives its unique Identity (`entity_id`), downloads the active scenario, and starts processing data.

```bash
# In a new terminal window:
./bin/fluxrig rack
```

**What just happened?**
1. The Rack initiated the **Deferred Adoption Lifecycle**.
2. Because the Mixer was running with `--auto-adopt`, the Rack was instantly approved.
3. The Rack downloaded its passport (`rack.flux`) and the `getting_started` scenario.
4. It started the local "Gears" (modules) defined in the scenario.

### Under the Hood: The Scenario Configuration

When the Rack connects to the Mixer, it receives the following declarative YAML logic. This scenario tells the Rack to deploy two **Bento Gears** connected by a **Wire**, demonstrating a complete message flow through the fluxrig topology.

```yaml
meta:
  name: Getting Started
  version: v1.0.0

# A generator gear produces synthetic telemetry data every 2 seconds,
# sends it through a wire to a sink gear, which logs to the console.

gears:
  - name: generator
    type: bento
    config:
      ports:
        outputs: ["out"]
      bento:
        input:
          generate:
            mapping: |
              root.timestamp = now()
              root.status = "UP"
              root.metrics.cpu_pct = random_int(min:10, max:85)
              root.metrics.mem_mb = random_int(min:512, max:4096)
              root.message = "Hello from fluxrig Getting Started!"
            interval: 2s
        # Output auto-wired via port "out"

  - name: sink
    type: bento
    config:
      ports:
        inputs: ["in"]
      bento:
        # Input auto-wired via port "in"
        output:
          stdout: {}

wires:
  - from: generator.out
    to: sink.in
```

**How it works:**
*   **`type: bento`**: We are defining declarative data-processing Gears without writing compiled Go code.
*   **`generator` gear**: Uses `input.generate` to produce synthetic JSON payloads every 2 seconds. Its output is auto-wired through the `out` port.
*   **`sink` gear**: Receives messages through the `in` port and prints them to the console via `output.stdout`.
*   **`wires`**: The `generator.out → sink.in` wire routes messages through the fluxrig bus (NATS JetStream), giving you full telemetry visibility: gear-level metrics like `flux.gear.messages_in` and `flux.gear.messages_out` are automatically tracked.
*   **`mapping`**: Uses Bloblang (Bento's mapping language) to inject mock metrics like a simulated CPU percentage (`cpu_pct`) and memory usage (`mem_mb`), alongside a timestamp. The message's own identity is not set here: fluxrig assigns every message a time-ordered UUID v7 `flux_id`, and the bridge reads one from message *metadata* rather than from the payload, so a `flux_id` written into the body is an ordinary field with a confusing name.

You should now see periodic logs in the Rack terminal reflecting the received data:
```text
{"flux_id":"...","metrics":{"cpu_pct":42,"mem_mb":1024},"message":"Hello from fluxrig Getting Started!"}
```

---

## 4. Verify Node Identity

Every Rack receives a **Sovereign Passport** upon enrollment. This binary file (`rack.flux`) contains the node's cryptographically signed identity and configuration. You can inspect this passport using the built-in security tools:

```bash
# Decode and verify the signed identity
./bin/fluxrig keys inspect data/rack.flux
```

**What you'll see:**
*   **MachineID**: The unique hardware identifier assigned to this node.
*   **Status**: The current lifecycle state (e.g., `active`).
*   **Revision**: How many times this identity has been re-issued.
*   **Signature Status**: The CLI automatically verifies that the identity hasn't been tampered with since issuance.

---

## 5. Observe the flow

The `getting_started` scenario automatically generates synthetic telemetry data (CPU/Memory metrics) on the Rack and streams it securely to the Mixer.

You can query this data directly using the `fluxrig` CLI!

Open a third terminal and query the real-time metrics:

```bash
# View the latest heartbeats sent by the Rack
./bin/fluxrig metrics --name flux.rack.heartbeats_sent

# View all metrics for your node (replace with your auto-generated node name if different)
./bin/fluxrig metrics --entity node-xxxxx
```

> [!NOTE]
> The Mixer exposes a REST API on port `8090` by default. The CLI is querying `http://localhost:8090/api/v1/telemetry/metrics` behind the scenes.

---

## 6. Next steps

Congratulations! You have successfully established a secure, bidirectional edge-to-cloud topology.

*   **Explore Scenarios**: Learn how to write your own data flows in the [Scenario Configuration](../reference/configuration.md) guide.
*   **Production Deployment**: Read about the [Security & PKI](../architecture/security.md) model to securely manage your cryptographic keys and disable `--auto-adopt`.
*   **ISO8583 Routing**: See the [Payments Tutorial](./iso8583_robot_suite.md) to route financial transactions.
