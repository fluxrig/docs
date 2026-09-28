---
slug: /reference/development/regression
title: E2E bash scripting
---

# E2E bash scripting

The **fluxrig** E2E suite collects shell scripts for **Infrastructure-Level Regression**. These tests check the platform's behavior regarding process lifecycle, operating system interactions, and network resiliency.

## Philosophy
Unlike functional tests that Robot Framework handles, E2E Bash tests treat the binary as a "Black Box." They focus on:

*   **Binary portability**: Ensure the Go binary runs on target distributions.
*   **Resource hygiene**: Check file descriptor management and memory usage.
*   **Signal handling**: Check graceful shutdown on `SIGTERM`.
*   **Network partitioning**: Test how the Rack recovers when NATS is unreachable.

## Suite structure
The suite lives in `test/e2e/`. It groups tests into a small number of segments. Each segment owns a Mixer lifecycle. A few suites each need a fully isolated Mixer of their own.

### Group A: one shared Mixer (`test/e2e/segments/group_a/`)
Nine checks run as sequential phases against one Mixer and its Racks, since none of them restart a process or need a conflicting Mixer setting:

| Phase | Description |
| :--- | :--- |
| Telemetry | Verification of DuckDB ingestion and OTel trace propagation. |
| Registration | Basic Rack/Mixer bootstrap and health check. |
| Conflict and identity | Identity collision handling (zero-downtime certificate rotation is **[Roadmap]**). |
| Data at rest | A message that crosses between two Racks leaves no readable card number on the Mixer or on either Rack, with the default settings and the default log level. |
| Load | High-level stress testing using the **[iso8583-tool](tools/iso8583_tool.md)**. |
| Spec and scenario manager | SDL validation and [content-addressable store](spec_manager.md#cas) integrity. |
| ISO 8583 universal matrix | Specialized validation for ISO 8583 binary protocol handling. |
| Processor simulator | A terminal drives a simulated processor over a real ISO 8583 socket, and receives an authored, correlated reply. |
| Payment switch | BIN-based routing across scheme uplinks, including concurrent multi-terminal load and a timeout-driven decline. |

### Group B: one shared Mixer (`test/e2e/segments/group_b/`)
Two checks share a Mixer that uses manual adoption (`auto_adopt = false`), since both need the pending-then-approve workflow:

| Phase | Description |
| :--- | :--- |
| Registry | Local topology validation and Mixer/Rack handshake. |
| CLI and admin | Exhaustive validation of admin CLI commands and filtering. |

### Isolated segments (`test/e2e/segments/<name>/`)
Each of these needs a Mixer or Rack restart as its own assertion. Or it needs a Mixer-wide setting that the groups above cannot share. Examples are TLS, no store encryption, and no automatic adoption:

| Segment | Description |
| :--- | :--- |
| `offline` | A Rack with a cached Passport starts while the Mixer is down, and finds it again when it returns. |
| `tls_simple` | Secure channel verification for Rack-to-Mixer links. |
| `io_tcp` | Low-level TCP framing and connection lifecycle validation across a Mixer and Rack restart. |
| `scenario_resume` | A Rack resumes its last scenario from its own copy after a restart, with the Mixer up, and starts it on its own when the Mixer is away. |
| `hot_lane` | A wire inside one Rack goes through memory: nothing of it reaches the Mixer, the Rack keeps serving it with the Mixer dead, and the same wire on the guaranteed lane does reach the Mixer. |
| `start_without_mixer` | A Rack starts and serves its saved scenario with no Mixer, and joins the Mixer when it returns without dropping a client that was connected. |
| `coatcheck` | State persistence and "resumable" transaction logic, including TTL expiry. |

### Suites that keep their own directory
Each of these has its own reason not to share a Mixer with anything else. It stays at `test/e2e/<name>/` rather than under `segments/`:

| Suite | Description |
| :--- | :--- |
| `19_card_data_iso` | A real ISO 8583 authorization, with the card number in DE 2, crosses between two Racks and is decoded on the way. The card number is in no file the Mixer or the Racks wrote: not in the message store, the logs, the DuckDB database or the Parquet files. Run at the info, debug and trace log levels. It reads DuckDB and Parquet through the `duckdb` command line tool, and checks first that its search sees a card number it was given on purpose. |
| `20_security_regression` | Authentication and secret-redaction checks against a Mixer that requires a real bearer token, standalone from the rest of the suite, which otherwise runs with authentication disabled for convenience. |
| `12_wasm_polyglot` | Wasm gear validation, run through Robot Framework rather than this bash harness, with its own Zig build step. |

## Running tests
Run the full regression suite with:

```bash
# Navigate to your local fluxrig repository root
cd path/to/fluxrig
make regression
```

`make regression` builds the binaries once. It then runs `test/e2e/run_consolidated.sh`. The script runs the segments above in parallel, a handful at a time to stay within one machine's resource budget. It reports a single pass or fail summary with a log file per segment.

### Running one segment
Each segment is self-contained:

```bash
bash test/e2e/segments/offline/run.sh
```

## Implementation notes
*   **Isolation**: Each segment runs on its own Mixer API port and Snake port, so segments never collide even when you start them together. Each starts its own temporary data directories.
*   **Assertions**: The suite checks with standard linux utilities (`grep`, `curl`, `jq`). It checks logs, API responses, and file contents.
*   **Artifacts**: A failed segment preserves its log and data directories for debugging. `run_consolidated.sh` also keeps every segment's full output in its own log file.
