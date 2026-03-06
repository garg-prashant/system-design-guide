# System Design: Log Aggregation

## Problem

Design a system to collect logs from many services, centralize them, store with retention, and enable search and analysis (e.g. ELK-style: Elasticsearch, Logstash, Kibana).

## Requirements

**Functional:** Ingest logs from many sources (files, stdout, network); store with retention; full-text search; filter by service, time, level; optional dashboards and alerts.

**Non-functional:** High ingestion throughput; low latency to search (seconds); durable storage; configurable retention; cost-effective at scale.

## High-Level Design

```
  Services (containers, VMs) → Shippers (Fluentd, Filebeat) → Message queue (Kafka) or direct
                                                                  ↓
  Indexer / Ingest → Storage (Elasticsearch, or object store + index) → Search API
                                                                              ↓
  Kibana / Dashboard, Alerting
```

- **Collection:** Agent on each host (or sidecar) reads log files or stdout; forwards to central queue or ingest endpoint. Optional: parse and structure (JSON) at agent.
- **Ingest:** Consume from queue; parse and enrich (add host, service); index in Elasticsearch (or write to object store and index separately). Bulk writes for throughput.
- **Storage:** Elasticsearch: full-text index; time-based indices (e.g. logs-2025-03-06); retention: delete old indices. Alternative: raw logs in S3; index metadata in search engine.
- **Search:** Query API (time range, query string, filters); Elasticsearch returns hits; paginate. Dashboards: saved queries and visualizations. Alerts: run query periodically; trigger on condition.

## Key Components

- **Shippers:** Lightweight; tail files or read stdout; send to Kafka or HTTP ingest. Backpressure and retry.
- **Message queue:** Kafka: buffer and decouple; partition by service or host; retention (e.g. 7 days) for replay.
- **Indexer:** Consume; parse (regex, JSON); map to schema; bulk index to Elasticsearch. Scale with partitions.
- **Elasticsearch:** Index per day or hour; shards; replication; delete by index for retention.
- **Search API:** REST; query DSL; auth and rate limit. Kibana: UI for search, dashboards, alerts.

## Data Model

- **Log document:** timestamp, level, message, service, host, trace_id, optional fields. Stored in Elasticsearch or similar.
- **Indices:** time-based (logs-YYYY-MM-DD); alias for “current” or “last 30 days.”

## Scale

- TB/day ingestion; index rate and search QPS. Scale Kafka partitions and indexer workers; Elasticsearch cluster by sharding and replication; cold data to object store + search on demand or separate tier.

## Trade-offs

- **Schema:** Structured (JSON) vs free text. Structured enables filtering and aggregation; free text needs parsing at query time.
- **Retention:** Hot (Elasticsearch) vs cold (S3). Hot for recent search; cold for compliance and cost.
- **Sampling:** At high volume, sample or aggregate to reduce cost; keep full logs for critical services.

## Failure & Operations

- **Backpressure:** Queue depth; slow down shippers or drop (with alert) if ingest can’t keep up.
- **Elasticsearch:** Replication for availability; monitor index lag and cluster health.
- **Monitoring:** Ingestion rate, index latency, search latency, storage growth, retention compliance.
