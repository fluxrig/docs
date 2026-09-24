---
slug: /reference/development/regression
title: E2E bash scripting
---

# E2E bash scripting

The **fluxrig** E2E suite is a collection of shell scripts designed for **Infrastructure-Level Regression**. These tests validate the platform's behavior regarding process lifecycle, operating system interactions, and network resiliency.

## Philosophy
Unlike functional tests (handled by Robot Framework), E2E Bash tests treat the binary as a "Black Box." They focus on:

*   **Binary portability**: Ensuring the Go binary runs on target distributions.
*   **Resource hygiene**: Verifying file descriptor management and memory usage.
*   **Signal handling**: Validating graceful shutdown on `SIGTERM`.
*   **Network partitioning**: Testing how the Rack recovers when NATS is unreachable.

## Suite structure
Tests are located in `test/e2e/` and organized by component:

| Suite | Description |
| :--- | :--- |
| `01_simple` | Basic Rack/Mixer bootstrap and health check. |
| `02_telemetry` | Verification of DuckDB ingestion and OTel trace propagation. |
| `03_registry` | Local topology validation and Mixer/Rack handshake. |
| `04_offline` | A Rack with a cached Passport starts while the Mixer is down, and finds it again when it returns. |
| `05_cli` | Exhaustive validation of admin CLI commands and filtering. |
| `06_conflict` | Identity collision handling (zero-downtime certificate rotation is **[Roadmap]**). |
| `07_load` | High-level stress testing using the **[iso8583-tool](tools/iso8583_tool.md)**. |
| `08_tls_simple` | Secure channel verification for Rack-to-Mixer links. |
| `09_io_tcp` | Low-level TCP framing and connection lifecycle validation. |
| `10_iso8583` | Specialized validation for ISO8583 binary protocol handling. |
| `11_coatcheck` | State persistence and "Resumable" transaction logic. |
| `12_specs` | SDL validation and [content-addressable store](spec_manager.md#cas) integrity. |
| `17_hot_lane` | A wire inside one Rack goes through memory: nothing of it reaches the Mixer, the Rack keeps serving it with the Mixer dead, and the same wire on the guaranteed lane does reach the Mixer. |
| `16_data_at_rest` | A message that crosses between two Racks leaves no readable card number on the Mixer or on either Rack, with the default settings and the default log level. |
| `18_start_without_mixer` | A Rack starts and serves its saved scenario with no Mixer, and joins the Mixer when it returns without dropping a client that was connected. |
| `19_card_data_iso` | A real ISO 8583 authorization, with the card number in DE 2, crosses between two Racks and is decoded on the way. The card number is in no file the Mixer or the Racks wrote: not in the message store, the logs, the DuckDB database or the Parquet files. Run at the info, debug and trace log levels. It reads DuckDB and Parquet through the `duckdb` command line tool, and checks first that its search sees a card number it was given on purpose. |
| `15_scenario_resume` | A Rack resumes its last scenario from its own copy after a restart, with the Mixer up, and starts it on its own when the Mixer is away. |

## Running tests
To execute the full regression suite:

```bash
# Navigate to your local fluxrig repository root
cd path/to/fluxrig
./test/e2e/run_all.sh
```

### Running individual tests
Each suite contains a `run.sh` script:

```bash
cd test/e2e/01_simple
./run.sh
```

## Implementation notes
*   **Isolation**: Each test suite typically spins up a dedicated NATS instance and temporary data directories to ensure a clean slate.
*   **Assertions**: Validation is performed using standard linux utilities (`grep`, `curl`, `jq`) checking logs, API responses, and file contents.
*   **Artifacts**: Failed tests preserve their log and data directories for debugging.
