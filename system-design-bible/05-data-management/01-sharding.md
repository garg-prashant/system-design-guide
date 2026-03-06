# Sharding (Horizontal Partitioning)

## What It Is

Split a dataset across **multiple independent databases** (shards). Each shard holds a subset of the data. No single node holds the full dataset; together they do.

```
  Application
       │
       ├──→ Shard 1 (users A–M)
       ├──→ Shard 2 (users N–Z)
       └──→ Shard 3 (users by region, etc.)
```

---

## Why Shard

- **Single DB can’t hold data or serve load** — disk, memory, or write throughput exceeds one machine.
- **Scale writes** — with single-leader replication, only primary takes writes; sharding spreads writes across shards.
- **Regulatory / locality** — keep data in a region (e.g. EU shard).

---

## Shard Key Selection

The **shard key** determines which shard a row lives on. Choice is critical; hard to change later.

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Range-based** | Key range per shard (e.g. user_id 0–1M, 1M–2M). | Range queries on key are efficient. | Hot spots if key is sequential (e.g. time); rebalancing can be heavy. |
| **Hash-based** | hash(shard_key) % N → shard. | Even distribution; no hot shard from key order. | Range queries on shard key need to hit all shards. |
| **Directory** | Lookup table: key → shard. | Flexible; can move keys for rebalancing. | Lookup table is a dependency; must be highly available. |

**Composite key:** e.g. (tenant_id, user_id) so all data for a tenant is co-located (multi-tenant) and you avoid cross-shard queries for that tenant.

---

## Challenges

- **Cross-shard queries:** No SQL JOIN across shards. Either denormalize, do application-level joins, or use a separate analytics store (e.g. replicated to data warehouse).
- **Cross-shard transactions:** No ACID across shards. Use saga or application-level consistency.
- **Rebalancing:** Adding/removing shards requires moving data; use consistent hashing to minimize movement, or double-write during migration.
- **Hot shards:** One shard key (e.g. celebrity user) gets most traffic; mitigate with sub-sharding, caching, or splitting that key.

---

## Interview Takeaways

1. Shard when **single DB can’t scale** (storage or write throughput).
2. **Shard key** drives distribution and query patterns; pick with range vs hash vs directory in mind.
3. **Cross-shard** = no JOINs, no global transactions; design APIs and data model accordingly.
4. **Consistent hashing** (from 03-distributed-systems-concepts) minimizes reassignment when shards are added/removed.
