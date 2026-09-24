---
slug: /reference/tech-stack
title: Technical stack
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Technical stack

Every dependency below is permissively licensed (Apache 2.0, MIT, BSD, ISC or MPL-2.0), which is a constraint this project holds rather than a preference: a copyleft dependency would reach the Apache 2.0 codebase.

> **Stand on the Shoulders of Giants**: **fluxrig** is proudly built upon the open source ecosystem. The projects below do the work this one is built on, and the communities maintaining them deserve the credit for it.

## Mixer & Rack (OSS)
*   **Language**: [Go (golang) 1.26+](https://go.dev/)
    - **Stability policy**: We adhere to the **level n version policy** (bleeding edge). We track the latest stable Go release.
    - **Rationale**: To support modern integrations (e.g., bento, Wasm) and leverage cutting-edge ergonomics (iterators), we accept the trade-off of being on the latest release.
    - **Deployment targets**: the **Rack** is pure Go and builds fully static with `CGO_ENABLED=0`, so it ships as a native Linux executable or in a `scratch` container. The **Mixer** requires **CGO** (the embedded DuckDB store), so it cannot be built with `CGO_ENABLED=0`.

### Rack build variants

The Rack ships in two variants. Binaries are stripped (`-ldflags "-w -s"`); sizes below are measured on the current release:

| Variant | Build | Size | Contents |
| :--- | :--- | :--- | :--- |
| **Full** | `make build` | **34 MB** | All built-in gears, including [`bento`](../reference/gears/bento.md) |
| **Lean** | `make build-lite` (`-tags nobento`) | **20 MB** | Everything except the `bento` gear |

The `bento` gear accounts for **14 MB (~43%)** of the full Rack, because it links Bento's engine plus protobuf, cue, avro and gojq transitively. Deployments that do not use `type: bento` can drop all of it with the lean build.

The tag also decides the licences in the binary. Bento reaches `hashicorp/golang-lru`, the only Mozilla Public License code in either binary; a lean build links 61 modules and every one of them is permissive.

> [!IMPORTANT]
> On a lean Rack, a scenario declaring `type: bento` fails at activation with `unknown gear type: bento`. This is deliberate and loud, rather than a silent no-op. Use `make build-all-variants` to produce both, and check which variant a binary is by the gear types it registers.

The **Mixer** is **105 MB**, dominated by the statically linked DuckDB engine, and **91 MB** with `-tags nobento`. It does link the gear factory, so the tag applies to it as well.

*   **Orchestration**: [Temporal.io](https://temporal.io/) (Go SDK) - **[Roadmap]**, for durable long-running business workflows. Scenario orchestration today is NATS subject-push from the Mixer, with no Temporal dependency.
*   **Wasm sandbox**: **[Roadmap]** Secure, polyglot execution of custom business logic in Rust, Go, or TypeScript. (The Wasm *filter* gear itself is already available, see the gear runtime below.)

*   **ISO8583 library**: [Moov-io/iso8583](https://github.com/moov-io/iso8583) (apache 2.0) - core parsing engine (used in both native core and Wasm gears).
*   **API architecture**: [cbor](https://cbor.org/) (data plane) + [Rest](https://restfulapi.net/) (control plane).
    - **Interface definition**: Go structs (serialized via cbor).
    - **Control plane**: Rest API (served by Mixer).
    - **Internal transport**: cbor messages over NATS (snake protocol).

*   **Messaging & streaming**:
    *   **Transport**: [NATS JetStream](https://nats.io/) ([Synadia](https://nats.synadia.com)) - chosen for high-performance, distributed persistence (snake protocol).
        - **Streams**: See **[wire protocols](../reference/protocols.md)** for the detailed subject topology (`flux.ctrl.>`, `flux.msg.>`, `flux.telemetry.>`).
    *   **Library**: [Watermill](https://watermill.io/) (mit) - Go library for building event-driven applications.
        - **Role**: Standardizes the "publisher/subscriber" interface. Current: **NATS** (durable). Planned: **gochannel** (RAM) for "turbo wire".
        - **Benefits**: Middleware (OTel, pivot tracing), mockability, and router pattern.

*   **CLI framework**: [Cobra](https://github.com/spf13/cobra) (apache 2.0) - standard library for building powerful CLI applications.
    *   **Key commands**: `Run`, `inspect config`, `inspect logs`, `logs`, `data query`

*   **Configuration**: [Koanf](https://github.com/knadh/koanf) (mit) - lightweight, extensible configuration management (toml/yaml/env) replacing viper.
*   **Logging**:
    *   **Library**: [Log/slog](https://pkg.go.dev/log/slog) (Go stdlib) - structured, high-performance logging.
    *   **Rotation**: [Lumberjack](https://github.com/natefinch/lumberjack) (mit).
    *   **Environment overrides**:
        *   `FLUXRIG_TRACE=1` - Override all log levels to `trace` (useful for debugging).
        *   `FLUXRIG_DISABLE_TELEMETRY=1` - Disable telemetry shipping.

*   **Gear runtime**: [Wazero](https://wazero.io/) (apache 2.0) - zero-dependency WebAssembly runtime for Go, enabling secure, platform-independent gear execution. Available today for the Wasm **filter** gear; polyglot **source** gears are **[Roadmap]**.

## Integration & ecosystem
*   **Universal I/O engine**: [Bento](https://github.com/warpstreamlabs/bento) (mit)
    - **Fork**: We explicitly use the **WarpStream Labs** fork (mit) to ensure permissive licensing, avoiding the Bento/Redpanda (BSL) restrictions.
    - **Role**: Provides the [bloblang](https://warpstreamlabs.github.io/bento/docs/guides/bloblang/about) mapping language and connectors. The shipped binary registers the pure and local I/O sets (file, csv, socket, websocket, HTTP, subprocess); the ecosystem's institutional connectors (Kafka, S3, AMQP) need a custom build.
    - **Integration**: Wrapped as a **native Gear** (`pkg/gears/native/bento`).

## Data & analytics
*   **Operational db**:
    *   **Standard tier**: [DuckDB](https://duckdb.org/) (embedded). high-performance OLAP + oltp for local state.
    *   **Enterprise**: [Distributed sql](https://en.wikipedia.org/wiki/Distributed_SQL) (external). supports ha, replication, and concurrency for Mixer state.

*   **Analytics dw**: [DuckDB](https://duckdb.org/) (mit) + [parquet](https://parquet.apache.org/) (apache 2.0) - high-performance OLAP on local storage.
*   **Visualization**:
    *   **Oss**: [Grafana](https://grafana.com/) - industry standard for observability dashboards.
    *   **Enterprise**: **Signoz** (open source apm) - evaluation candidate (full-stack apm & tracing ui).

*   **Secure store**:
    *   **OSS**: AES-256-GCM (software encryption).
    *   **Enterprise**: [HashiCorp Vault](https://developer.hashicorp.com/vault) or [AWS KMS](https://aws.amazon.com/kms/).

## Scenario visualization & management UI
*   **Topology model & viewer**: [LikeC4](https://likec4.dev/) (mit) - architecture-as-code model with interactive drill-down diagrams. The CLI generates a model from any scenario (`fluxrig scenario viz`); viewing uses the likec4 toolchain at development time, never in the shipped binaries.
*   **Console framework** `[Roadmap]`: [React](https://react.dev/) (mit) + [Vite](https://vite.dev/) (mit) - the operation console builds on `@likec4/diagram`, which renders via [React flow](https://reactflow.dev/) (mit), and ships embedded inside the Mixer binary (no external toolchain at runtime).
*   **Code editor** `[Roadmap]`: [Monaco editor](https://microsoft.github.io/monaco-editor/) (mit) - embedded for editing specs and scenarios with server-side validation.

## AI & analytics (local) (future / research)
*   **Inference engine**: [ONNX Runtime](https://onnxruntime.ai/) / [llama.cpp](https://github.com/ggerganov/llama.cpp) - for running AI models locally within the container.
*   **Models**:
    *   [Scikit-Learn](https://scikit-learn.org/) (BSD) (Isolation Forest) for Anomaly Detection.
    *   Quantized LLMs (e.g., Llama-3) (Community License) for the Natural Language Assistant.


## Observability, tracing & ID generation

**fluxrig** uses a simplified observability strategy based on **OpenTelemetry** and the **DuckLake** pattern:

| Tier | Backend(s) | Scale | Phase |
| :--- | :--- | :--- | :--- |
| **Embedded** | DuckDB + Parquet (S3/Local) | < 100M events/day | **Stable** (current release) |
| **Standard** | Arc + OpenSearch | Team deployments | **[Roadmap]** |
| **Enterprise** | ClickHouse + SigNoz | Petabyte scale | **[Roadmap]** |


| Category | Tool/Library | Type | Purpose |
| :--- | :--- | :--- | :--- |
| **Telemetry SDK** | **[OpenTelemetry Go](https://opentelemetry.io/)** (Apache 2.0) | Library | Metrics, Logs, & Distributed Tracing. |
| **Query Engine** | **[DuckDB](https://duckdb.org/)** (MIT) | OLAP | Stateless SQL engine for Metrics & Traces (Embedded). |
| **Query Lang** | **[PRQL](https://prql-lang.org/)** (Apache 2.0) | Language | Time-series optimized query language (compiles to SQL). |
| **Logs** | **[OpenSearch](https://opensearch.org/)** (Apache 2.0) | Search | Evaluation Candidate for Log Search. |
| **Analytics** | **[ClickHouse](https://clickhouse.com/)** (Apache 2.0) | Database | Evaluation Candidate for Petabyte-scale OLAP (Enterprise). |
| **ID Generation** | **[Google/uuid](https://github.com/google/uuid)** (BSD) | Library | Distributed 128-bit UUID v7 (RFC 9562) |
| **OTLP Export** | **[OTLP](https://opentelemetry.io/docs/specs/otlp/)** | Standard | **[Roadmap]** Vendor-neutral export to any OTel-compatible backend. Not built today. |

> **Ref**: See [Observability Architecture](../architecture/observability.md) for detailed tier documentation.

## Testing, security & compliance

> **Security First**: **fluxrig** development follows strict security standards (PCI-DSS, OWASP, SSF) from the very beginning. All architectural decisions, dependency choices, and coding practices are audited to ensure compliance.

*   **Testing frameworks**:
    *   **[Robot framework](https://robotframework.org/)** (Apache 2.0): Keyword-driven testing for E2E and Acceptance testing.
    *   **[Python](https://www.python.org/)** (PSF): Scripting language for advanced test scenarios and the **fluxrig** SDK.
    *   **[Testcontainers](https://testcontainers.com/)** (MIT): Ephemeral Docker containers for integration testing.

*   **Static analysis (SAST)**:
    *   **[Gosec](https://github.com/securego/gosec)** (Apache 2.0): Go security checker (SQLi, credentials, etc.).
    *   **[Govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)** (BSD): Go vulnerability database checker.
    *   **[Bandit](https://github.com/pycqa/bandit)** (Apache 2.0): Python security linter.

*   **Container & secret security**:
    *   **[Trivy](https://github.com/aquasecurity/trivy)** (Apache 2.0): Comprehensive container and filesystem vulnerability scanner.
    *   **[Trufflehog](https://github.com/trufflesecurity/trufflehog)** (AGPL - External CLI): Scans git history for high-entropy secrets.

*   **Code coverage**:
    *   **Go cover**: Built-in `Go test -coverprofile` for tracking unit test coverage.
    *   **Standards**:
        *   **Core Libraries (`pkg/*`)**: **> 70%** required.
        *   **Overall project**: **> 60%** required.
    *   **Coverage.py**: Code coverage measurement for Python scripts.

## Utility libraries

**fluxrig** also utilizes these key utility libraries:

*   **[Gin](https://github.com/gin-gonic/gin)**: High-performance http web framework.
*   **[Google/uuid](https://github.com/google/uuid)**: Robust uuid generation.
*   **[Gopkg.in/yaml.v3](https://github.com/go-yaml/yaml)**: Strict yaml marshaling/unmarshaling.

## Documentation & build

The documentation site and PDF generation are powered by:

*   **[Docusaurus](https://docusaurus.io/)**: Modern static site generator (Web version).
*   **[MkDocs](https://www.mkdocs.org/)**: Static site generator (PDF Core version).
*   **[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/)**: Theming and UX for the PDF pipeline.
*   **[Mike](https://github.com/jimporter/mike)**: Versioning management for MkDocs (git-based).
*   **Docusaurus OpenAPI**: Modern OpenAPI reference renderer (Scalar/Redocusaurus).
*   **[Mkdocs-with-pdf](https://github.com/orzih/mkdocs-with-pdf)**: PDF generation plugin (using **weasyprint**).
*   **[Mermaid](https://mermaid.js.org/)**: Diagramming and visualization (rendered via `mmdc` CLI).
*   **[Python](https://www.python.org/)**: Pre-processing scripts for validation and layout.