# System Design: Real-Time Analytics

## Problem

Design a system to ingest high-volume events (e.g. clicks, page views), process them in real time (aggregations, filters, joins), and serve low-latency queries (dashboards, alerts). Think stream processing (Flink, Kafka Streams) and real-time OLAP.

## Requirements

**Functional:** Ingest events (append-only stream); run aggregations (count, sum, windowed); filter and transform; join streams (optional); query results (last N minutes, by dimension); alert on thresholds.

**Non-functional:** Low ingestion latency (seconds to queryable); high throughput (millions of events/s); exactly-once or at-least-once processing; scalable and fault-tolerant.

## High-Level Design

```
  Events → Ingestion (API / Kafka) → Stream processor (Flink, etc.) → Sinks
                ↓                           ↓
  Windowing (tumbling, sliding)      Aggregations (count, sum by key)
                ↓                           ↓
  State store (RocksDB, etc.)   →  Output: DB, cache, or stream (query API)
```

- **Ingestion:** Events (e.g. user_id, event_type, timestamp, dimensions) to Kafka (or Kinesis); partition by key for ordering; retention (e.g. 7 days).
- **Stream processing:** Consume from Kafka; key by (user, dimension); window (e.g. 1-min tumbling, 5-min sliding); aggregate (count, sum); emit to sink. State: store window state for fault tolerance (checkpoints).
- **Sinks:** Write to key-value store (Redis) or time-series DB for real-time query; or to data lake/warehouse for batch + real-time hybrid.
- **Query API:** Query by time range and dimensions; read from Redis or time-series DB; optional SQL on warehouse for historical.

## Key Components

- **Message queue:** Kafka; partition by event key; retention for replay and backfill.
- **Stream processor:** Flink, Kafka Streams, or Kinesis Data Analytics; windows (tumbling, sliding, session); aggregations; exactly-once with checkpointing and transactional sink.
- **State:** Processor state (window buffers, counters); backed by RocksDB or similar; checkpoint to durable storage for recovery.
- **Sink:** Redis (key = dimension + window → count); or time-series DB; or warehouse (batch upsert). Query layer reads from sink store.
- **Alerting:** Rule engine on stream or on query result (e.g. count > threshold in 5-min window); trigger notification.

## Data Model

- **Event:** event_id, user_id, event_type, timestamp, dimensions (key-value). Schema or schema-on-read.
- **Aggregation output:** (dimension_keys, window_start, window_end) → count, sum, etc. Stored in sink.

## Scale

- Millions of events/s; partition Kafka and processor parallelism; state per key; scale processors horizontally. Sink: Redis cluster or time-series DB; query layer caches hot results.

## Trade-offs

- **Exactly-once vs at-least-once:** Exactly-once: checkpointing + idempotent sink; more overhead. At-least-once: simpler; dedup at query or accept approximate.
- **Window type:** Tumbling (fixed windows) vs sliding (overlapping). Sliding: more accurate for “last 5 min”; more state.
- **Query store:** Redis for sub-second; time-series for range queries; warehouse for historical; hybrid common.

## Failure & Operations

- **Backpressure:** If sink is slow, backpressure to Kafka; monitor lag and scale processors/sink.
- **Checkpointing:** Flink checkpoints; recover from last checkpoint on failure; replay from Kafka for exactly-once.
- **Monitoring:** Ingestion rate, processing lag, sink write latency, query latency, alert delivery.
