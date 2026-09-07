---
slug: /reference/development/iso8583-tool
title: ISO8583 utility
---

# ISO8583 utility

A unified tool for ISO8583 testing, combining a Load Generator and an Echo Server.

## Modes

The tool operates in two modes:

1. **Load mode** (`-mode load`): Generates ISO8583 traffic with embedded RTT metrics.
2. **Echo mode** (`-mode echo`): Starts a transparent TCP server that echoes (or discards) raw bytes. This mode is the standard component for the **Loopback Endpoint** in the **[Verification Rig](../../architecture/testing_simulation.md#scenario-driven-load-testing)**.

## Usage

```bash
iso8583-tool -mode [load|echo] [flags]
```

### Load mode

Generates traffic to a target server.

```bash
iso8583-tool -mode load \
  -target localhost:8583 \
  -concurrency 10 \
  -rate 100 \
  -duration 30s
```

#### Flags (load)

| Flag | Default | Description |
|------|---------|-------------|
| `-target` | `127.0.0.1:8583` | Target ISO8583 server address |
| `-concurrency` | `10` | Number of parallel TCP connections |
| `-rate` | `100` | Target total transactions per second |
| `-duration` | `30s` | Test duration |
| `-warmup` | `0` | Warmup duration before measurement (e.g., `2s`) |
| `-encoding` | `bcd` | ISO8583 encoding: `ascii` or `bcd` |
| `-framing-bytes` | `2` | Length prefix size: `2` or `4` bytes |
| `-header-len` | `12` | Correlation header length (min 8 for RTT) |
| `-report` | `report.json` | JSON report output file |

### Advanced load scenarios

#### Static header (simulation)

Simulate a specific variant header (e.g., Visa V.I.P routing) by injecting a static hex string.

> **Note**: RTT metrics are **disabled** when using a static header, as the tool cannot inject the timestamp.

```bash
# Send "0800" MTI + 22-byte V.I.P. header
./bin/iso8583-tool -mode load -header-val "16010203000000000000000000000000000000000000"
```

### Echo mode

Starts a backend simulation server.

```bash
iso8583-tool -mode echo -port 8590
```

#### Flags (echo)

| Flag | Default | Description |
|------|---------|-------------|
| `-port` | `8590` | Port to listen on |
| `-sink` | `false` | If true, discards data instead of echoing (Blackhole) |

## Metrics & reports

The Load Mode generates a comprehensive JSON report including:

- **Global stats**: Total sent/received, TPS, RPS.
- **Latency percentiles**: P50, P90, P99, max.
- **Per-connection stats**: Detailed breakdown for each TCP connection.
- **Time series**: Per-second metrics buckets.

### Header format (RTT)

When RTT is enabled (`-header-len >= 8`), the tool prepends a header:

```
[8-byte timestamp][2-byte conn_id][2-byte padding]...
```

This allows for stateless latency measurement and connection tracking.

## Performance benchmarking

A pre-packaged benchmark script is available to validate the limits of your local environment or infrastructure.

```bash
# Location: test/performance/benchmark.sh
./test/performance/benchmark.sh
```

The benchmark runs three scenarios:

1.  **Baseline**: Warm-up and basic verification.
2.  **Concurrency stress**: Tests OS file descriptor and scheduler limits (high concurrency).
3.  **Endurance**: Sustained high-throughput test (5000+ TPS) for 2 minutes.