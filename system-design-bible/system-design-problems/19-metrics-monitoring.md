# System Design: Metrics Monitoring

## Problem

Design a metrics and monitoring system: collect time-series metrics from many services (CPU, latency, custom counters), store them, aggregate and query, and alert when thresholds are breached (e.g. Prometheus + Grafana style).

## Requirements

**Functional:** Scrape or receive metrics (counters, gauges, histograms); store time-series; query (instant, range, aggregation); dashboards; alerting (rules and notifications).

**Non-functional:** High write throughput; low query latency; retention (e.g. 15d hot, 1y cold); high availability for alerting.

## High-Level Design

```
  Services (metrics endpoint) → Scrapers / Remote write → Time-series DB (Prometheus, M3, etc.)
                                                                  ↓
  Query API → Dashboards (Grafana)  |  Alert manager → Evaluate rules → Notify (PagerDuty, Slack)
```

- **Collection:** Pull (scraper hits /metrics) or push (services push to gateway); metrics with labels (job, instance, env). Remote write or native format.
- **Storage:** Time-series DB: append-only; compress; downsampling for long retention (e.g. 1m → 5m for old data). Index by metric name and labels.
- **Query:** Instant (single timestamp), range (time range), aggregation (sum, rate, avg by label). PromQL or similar.
- **Alerting:** Rules (e.g. rate(errors[5m]) > 0.01); evaluate periodically; fire alerts to alert manager; route to PagerDuty/Slack; dedupe and silence.

## Key Components

- **Scraper / Push gateway:** Discover targets (or accept push); fetch metrics; remote write to storage. Optional: federation for hierarchy.
- **Time-series DB:** Prometheus TSDB, M3, VictoriaMetrics, or cloud (e.g. CloudWatch, Datadog). Write path: append; read path: query with aggregation.
- **Query API:** HTTP; PromQL or GraphQL; used by Grafana and alert manager.
- **Alert manager:** Load rules; evaluate; group and route alerts; notify; dedupe; silence.
- **Dashboards:** Grafana (or similar); connect to query API; visualizations and variables.

## Data Model

- **Metric:** name, labels (key=value), type (counter/gauge/histogram); samples (timestamp, value). Stored as time-series.
- **Rules:** name, expr (PromQL), for (duration), labels, annotations. Stored in config or DB.

## Scale

- Millions of time-series; hundreds of thousands of samples/s. Storage: compression and downsampling; cluster or shard by metric/label. Query: limit range and series; cache where possible.

## Trade-offs

- **Pull vs push:** Pull: simple, scraper discovers; push: good for short-lived jobs (batch). Both supported in many systems.
- **Cardinality:** High cardinality (e.g. user_id in label) blows up storage and query; avoid or limit.
- **Retention:** Hot (raw) vs cold (downsampled); trade cost vs granularity for old data.

## Failure & Operations

- **Availability:** Replicate TSDB; alert manager HA; avoid single point of failure for alert evaluation.
- **Backfill:** Support backfill for recovery; careful with resource usage.
- **Monitoring:** Scrape success, write latency, query latency, alert delivery, cardinality growth.
