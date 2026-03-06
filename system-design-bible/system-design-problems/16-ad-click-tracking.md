# System Design: Ad Click Tracking

## Problem

Design a system to track ad clicks at scale: record every click (user, ad, timestamp, device, etc.), support real-time and batch analytics, and enable fraud detection and billing.

## Requirements

**Functional:** Record click events (ad_id, user_id, timestamp, IP, user_agent, etc.); query by campaign/ad for aggregates; near real-time dashboards; fraud detection (optional); billing reconciliation.

**Non-functional:** Very high write throughput (millions of clicks/s); durable (no loss for billing); low latency for ingestion; scalable storage and aggregation.

## High-Level Design

```
  Click events → API / SDK → Message queue (Kafka/Kinesis) → Stream processors
                                                                  ↓
  Real-time: Aggregation (per ad/campaign, window) → Time-series or cache → Dashboard
  Batch: Data lake / warehouse → ETL → Aggregates, billing, fraud
```

- **Ingestion:** Accept click event; validate; push to message queue (async); respond 200 quickly. Queue buffers and decouples from processors.
- **Stream processing:** Consume from queue; aggregate by (ad_id, campaign_id, window); write to time-series DB or key-value store for real-time dashboards.
- **Batch:** Same events (or copy) to data lake (S3 + partition by date); batch jobs for hourly/daily aggregates, billing, fraud rules.
- **Query:** Real-time: read from time-series or cache. Historical: query warehouse or pre-aggregated tables.

## Key Components

- **Ingestion API:** Stateless; validate and enqueue; idempotency key to deduplicate (same click_id = one count). Partition by ad_id or user_id for ordering if needed.
- **Message queue:** Kafka or Kinesis; partition for parallelism; retention (e.g. 7 days) for replay and batch.
- **Stream processor:** Flink, Spark Streaming, or Kinesis consumer; windowed aggregation (e.g. clicks per ad per minute); write to Redis/DB or time-series.
- **Time-series / OLAP:** Time-series DB (e.g. InfluxDB, TimescaleDB) or columnar warehouse (BigQuery, Redshift) for batch.
- **Fraud:** Rules (same IP many clicks, bot pattern) or ML; run in stream or batch; flag or filter for billing.

## Data Model

- **Click event:** click_id, ad_id, campaign_id, user_id, timestamp, ip, user_agent, device_id. Optional: idempotency_key.
- **Aggregates:** (ad_id, window) → count, unique_users (approx). Stored in time-series or cache.
- **Billing:** Pre-aggregated by campaign, date; reconciled with ad server logs.

## Scale

- Millions of clicks per second; ingestion must not drop events. Queue partitions scale with consumers; stream processors scale horizontally; storage: time-series and data lake scale with retention and partitioning.

## Trade-offs

- **Exactly-once:** Idempotency key at ingestion; exactly-once processing with transactional sink or idempotent writes.
- **Real-time vs batch:** Real-time for dashboards; batch for billing and fraud (consistency, complex logic).
- **Storage:** Hot (recent) in time-series or cache; cold in data lake; tier by age.

## Failure & Operations

- **Queue:** Replication and retention; backpressure if consumers lag; alert on lag.
- **Durability:** Ack only after event is durable in queue; processors commit offsets after write.
- **Monitoring:** Ingestion rate, queue lag, aggregation latency, billing discrepancy alerts.
