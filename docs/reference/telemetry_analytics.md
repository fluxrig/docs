---
slug: /reference/operations/telemetry
title: Telemetry & analytics
---

# Telemetry & analytics

This section provides advanced query patterns for analyzing **fluxrig** observability data across different storage tiers.

In the **Embedded Tier** (the tier implemented in the current release), all telemetry (traces, logs, metrics) and business messages are stored as **Partitioned Parquet** files. This "Cold Storage" strategy provides institutional audit readiness without the overhead of a centralized database, while remaining queryable via the **Operational Ledger** (DuckDB).

## Warehouse structure
Data is automatically exported from the Rack's active buffers to the local Warehouse using an hourly partitioning scheme:

- **Logs**: `data/telemetry/logs/YYYY/MM/DD/HH/logs_<timestamp>.parquet`
- **Metrics**: `data/telemetry/metrics/YYYY/MM/DD/HH/metrics_<timestamp>.parquet`
- **Messages**: `data/messages/<wire_id>/YYYY/MM/DD/HH/messages_<wid>_<timestamp>.parquet`

## Hybrid analysis
The platform leverages DuckDB's `read_parquet` capabilities discoverable via the Registry to join active state with the hourly archives.

```sql
-- Analyze errors by Gear across active memory AND cold storage
SELECT gear_id, count(*) as error_count
FROM (
  SELECT gear_id, level FROM active_logs
  UNION ALL
  SELECT gear_id, level FROM read_parquet('./data/telemetry/logs/**/*.parquet')
)
WHERE level = 'error' AND ts > NOW() - INTERVAL '4 hours'
GROUP BY 1 ORDER BY 2 DESC;
```

## Business intelligence (fluxspec)
Analyzing promoted fields from business messages.

```sql
-- Analyze transactions by BIN and calculate totals
SELECT bin, currency, SUM(amount) as total
FROM './data/messages/fluxSpec/visa-v1/**/*.parquet'
WHERE ts > NOW() - INTERVAL '24 hours'
GROUP BY 1, 2 ORDER BY total DESC;
```

---

## Real-time metrics (Prometheus / OTel)

For DevOps and SREs requiring real-time dashboarding and alerting, **fluxrig** natively exposes metrics compatible with OpenTelemetry and Prometheus.

### Prometheus scraping
If utilizing standard Prometheus scraping, you can extract core runtime telemetry (Goroutines, Memory, GC cycles) and Gear throughput.

Endpoint (Mixer & Rack):
```
GET /metrics
```
*Port is dependent on the configured API port (default 8090 for Mixer).*

### Key metrics to monitor
* `flux.gear.messages_in`: Throughput capacity (entering gears).
* `flux.gear.messages_out`: Throughput capacity (leaving gears).
* `flux.gear.processing_time_ms`: Gear execution latency histogram.
* `flux.port.bytes_in`: Ingress data volume at the port level.
* `flux.port.bytes_out`: Egress data volume at the port level.
* `flux.nats.publish_latency_ms`: Message bus propagation health.
* `flux.bus.publish_errors`: System-level emission failures.

---

### Specialized logging: the TRACE level
For deep protocol inspection and high-volume signal debugging, `fluxrig` implements a custom **`TRACE`** log level (`slog -8`).

*   **Role**: Used for full bit-perfect dumps of incoming/outgoing payloads and complex dialect parsing results.
*   **Usage**: Activate via the `--level trace` flag in the Rack or via the Mixer API.

```bash
# Start a rack with per-message tracing
fluxrig rack --level trace
```

> [!CAUTION]
> **Performance Impact**: Activating `TRACE` level on production high-frequency gears (e.g., 1000+ tps) can generate gigabytes of logs per minute. It should be used surgically for diagnostic sessions.

---

## OpenSearch analytics (enterprise tier)

For high-volume Enterprise deployments, logs are indexed in **OpenSearch** for full-text search and complex aggregations.

### Search patterns
Using the OpenSearch DSL to find specific events.

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "timeout" } },
        { "range": { "ts": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

---

## Data schema reference

| Field | Type | Description |
| :--- | :--- | :--- |
| `ts` | Timestamp | Event generation time |
| `trace_id` | String | W3C Correlation ID |
| `flux_id` | UUID | UUID v7 Business ID |
| `entity_name` | String | Name of the Rack or Mixer |
| `gear_id` | String | ID of the Gear that generated the signal |
| `level` | String | Log level (info, warn, error, debug) |

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
