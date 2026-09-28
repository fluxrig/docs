---
slug: /tutorials/quickstart
title: 5-Minute Quickstart
---

# 5-minute quickstart

This guide shows a functional local deployment of **fluxrig** in under 5 minutes. Start the control plane. Enroll an edge node. Process telemetry data without writing custom code or configuring complex registries.

> [!TIP]
> **Zero config first-run**: By default, fluxrig automatically generates necessary cryptographic keys and embedded database files in your working directory. You do not need a pre-existing complex database or secret manager setup to start!

## Prerequisites

*   **Linux or macOS**
*   **Go 1.26+** (if compiling from source)
*   **Make**

---

## 1. Get the binaries

Currently, building from source is the only available option. Clone the repository. Run `make build`:

```bash
git clone https://github.com/jaab-tech/fluxrig.git
cd fluxrig
make build
```

This will produce the two core binaries in the `./bin` directory:
*   `fluxrig-mixer`: The centralized Control Plane (Registry, Telemetry, NATS Bus).
*   `fluxrig`: The unified CLI and Edge Node (The Rack).

---

## 2. Start the Mixer (control plane)

The Mixer acts as the brain of your rig. It manages edge node enrollment. It maintains the system topology. It ingests telemetry.

For this quickstart, we will use the built-in `getting_started.yaml` scenario and the `--auto-adopt` flag. The `--auto-adopt` flag bypasses the manual approval step for new edge nodes. This helps local development.

```bash
# In your first terminal window:
./bin/fluxrig-mixer --auto-adopt examples/scenarios/getting_started.yaml
```

**What just happened?**
1. The Mixer initialized an embedded **DuckDB** telemetry store.
2. It started an embedded **NATS JetStream** server on port `4222`.
3. It imported the `getting_started.yaml` scenario and set it to active.

---

## 3. Start a Rack (edge node)

The Rack is the edge execution node. It connects to the Mixer. It receives its unique Identity (`entity_id`). It downloads the active scenario. It starts processing data.

```bash
# In a new terminal window:
./bin/fluxrig rack
```

**What just happened?**
1. The Rack started the **Deferred Adoption Lifecycle**.
2. Because the Mixer ran with `--auto-adopt`, it approved the Rack instantly.
3. The Rack downloaded its passport (`rack.flux`) and the `getting_started` scenario.
4. It started the local "Gears" (modules) defined in the scenario.

### Under the hood: the scenario configuration

When the Rack connects to the Mixer, it receives the following declarative YAML logic. This scenario tells the Rack to deploy two **Bento Gears** with a **Wire**. This shows a complete message flow through the fluxrig topology.

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
*   **`type: bento`**: This defines declarative data-processing Gears without compiled Go code.
*   **`generator` gear**: The `generator` gear uses `input.generate` to produce synthetic JSON payloads every 2 seconds. The setup auto-wires its output through the `out` port.
*   **`sink` gear**: The `sink` gear receives messages through the `in` port. It prints them to the console via `output.stdout`.
*   **`wires`**: The `generator.out → sink.in` wire routes messages through the fluxrig bus (NATS JetStream). This gives telemetry visibility. The setup automatically tracks gear-level metrics like `flux.gear.messages_in` and `flux.gear.messages_out`.
*   **`mapping`**: The `mapping` uses Bloblang (Bento's mapping language) to inject mock metrics. It injects a simulated CPU percentage (`cpu_pct`) and memory usage (`mem_mb`) with a timestamp. The setup does not set the message's own identity here. fluxrig assigns every message a time-ordered UUID v7 `flux_id`. The bridge reads one from message *metadata* rather than from the payload. Writing a `flux_id` into the body creates an ordinary field with a confusing name.

You should now see periodic logs in the Rack terminal reflecting the received data:
```text
{"flux_id":"...","metrics":{"cpu_pct":42,"mem_mb":1024},"message":"Hello from fluxrig Getting Started!"}
```

---

## 4. Verify node identity

Every Rack receives a **Sovereign Passport** upon enrollment. This binary file (`rack.flux`) contains the node's cryptographically signed identity and configuration. You can inspect this passport using the built-in security tools:

```bash
# Decode and verify the signed identity
./bin/fluxrig keys inspect data/rack.flux
```

**What you will see:**
*   **MachineID**: The unique hardware identifier assigned to this node.
*   **Status**: The current lifecycle state (e.g., `active`).
*   **Revision**: How many times the Mixer re-issued this identity.
*   **Signature Status**: The CLI automatically verifies that the identity has not been tampered with since issuance.

---

## 5. Observe the flow

The `getting_started` scenario automatically generates synthetic telemetry data (CPU/Memory metrics) on the Rack. It streams it securely to the Mixer.

You can query this data directly with the `fluxrig` CLI.

Open a third terminal. Query the real-time metrics:

```bash
# View the latest heartbeats sent by the Rack
./bin/fluxrig metrics --name flux.rack.heartbeats_sent

# View all metrics for your node (replace with your auto-generated node name if different)
./bin/fluxrig metrics --entity node-xxxxx
```

> [!NOTE]
> The Mixer exposes a REST API on port `8090` by default. The CLI queries `http://localhost:8090/api/v1/telemetry/metrics` behind the scenes.

---

## 6. Next steps

Congratulations. You established a secure, bidirectional edge-to-cloud topology.

*   **Explore Scenarios**: Learn how to write your own data flows in the [Scenario Configuration](../reference/configuration.md) guide.
*   **Production Deployment**: Read about the [Security & PKI](../architecture/security.md) model to securely manage your cryptographic keys and disable `--auto-adopt`.
*   **ISO8583 Routing**: See the [Payments Tutorial](./iso8583_robot_suite.md) to route financial transactions.
