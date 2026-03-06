# Indexing, OLTP, and OLAP

## Indexing

**Purpose:** Speed up reads by maintaining a secondary structure that allows fast lookup (e.g. by key, range, or full-text).

| Index type | Use case | Trade-off |
|------------|----------|-----------|
| **B-tree** | Default in most RDBMS; range and equality. | Writes pay cost to update index; storage overhead. |
| **Hash** | Equality only (e.g. key-value). | No range scans; good for point lookups. |
| **Secondary index** | Lookup by non-primary column. | In distributed DB, can require scatter-gather or global index. |
| **Composite** | Multiple columns (e.g. (tenant_id, created_at)). | Order of columns matters for query coverage. |
| **Full-text** | Search in text (inverted index). | Different store (e.g. Elasticsearch) or DB extension. |

**Interview:** “We index columns used in WHERE and ORDER BY; we avoid over-indexing to keep writes fast and storage low.”

---

## OLTP vs OLAP

| | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|--------------------------------------|-------------------------------------|
| **Workload** | Short transactions; many small reads/writes; high concurrency. | Complex queries; large scans; aggregations; reporting. |
| **Data** | Current, normalized; row-oriented. | Historical, denormalized; often columnar. |
| **Examples** | User service, orders, inventory. | Dashboards, analytics, data warehouse. |
| **Scale** | Latency and throughput per transaction. | Throughput for large scans; query time. |

**Pattern:** OLTP DBs feed (via ETL/CDC) into OLAP/data warehouse. Don’t run heavy analytics on the OLTP DB—offload to a dedicated store.

---

## When to Use What

- **Key-value / document (e.g. DynamoDB, MongoDB):** High throughput, flexible schema, simple access by key or simple query.
- **Relational (e.g. PostgreSQL):** ACID, JOINs, complex queries within one DB.
- **Search (e.g. Elasticsearch):** Full-text, faceted search, log search.
- **Time-series:** Metrics, events; append-heavy; time-based retention and aggregation.
- **Columnar / warehouse (e.g. BigQuery, Snowflake, Redshift):** Analytics, large scans, aggregations; batch or interactive.

**Interview:** “We keep transactional workload on OLTP; we replicate or stream to a data warehouse for OLAP so reporting doesn’t impact production.”
