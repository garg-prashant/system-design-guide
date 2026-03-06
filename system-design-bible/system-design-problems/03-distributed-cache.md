# System Design: Distributed Cache

## Problem

Design a distributed cache that multiple application servers can use to store and retrieve key-value data. It should scale horizontally, handle node failures, and minimize cache misses and duplication.

## Requirements

**Functional:** Get(key), Set(key, value), Delete(key); optional TTL. Multiple clients; shared view of data (or partitioned).

**Non-functional:** Low latency (sub-ms for gets); high throughput; availability when some nodes fail; minimal data movement when nodes are added/removed.

## High-Level Design

```
  App servers → Cache client (consistent hash) → Cache nodes (e.g. N nodes)
       ↓
  Each key hashed to one (or a few) nodes; replication for fault tolerance
```

- **Partitioning:** Keys distributed across nodes (e.g. consistent hashing so only 1/N keys move when a node is added).
- **Replication:** Each key stored on K nodes (e.g. 2) for availability; read from any replica; write to all replicas (or primary).

## Key Components

- **Consistent hashing:** Ring of hash space; each node has virtual nodes on ring; key hashes to ring position → responsible node(s). Adding/removing node affects only adjacent keys.
- **Eviction:** Per-node LRU (or LFU) when memory limit reached; TTL expiry.
- **Client:** Knows ring (node list); hashes key → node; sends request to that node. On failure, retry or use replica.
- **Replication:** Primary-replica or multi-primary; conflict resolution if multi-write (e.g. last-write-wins, version vector).

## Data Model

- **Key-value:** Opaque key and value; size limits (e.g. key 256 B, value 1 MB). Optional: key namespace, TTL.
- **Metadata:** TTL, version (if needed for invalidation).

## Scale

- Assume 1M keys, 1 KB avg value → 1 GB per node; 10 nodes → 10 GB total. 100K reads/s → each node ~10K reads/s; in-memory can do 50K–100K/s per node.
- Add nodes to increase capacity and throughput; consistent hashing minimizes reassignment.

## Trade-offs

- **Consistent hashing vs modulo:** Consistent hashing avoids mass reassignment on topology change; modulo is simpler but all keys can shift.
- **Replication factor:** Higher K = more availability and read capacity; more storage and write cost.
- **Cache-aside vs read-through:** Cache-aside: app manages population; read-through: cache layer fetches on miss (simpler app, cache must know origin).

## Failure & Operations

- **Node failure:** Replicas serve reads; rebalance to restore replication. Client retries another replica.
- **Thundering herd:** Many requests miss same key → all hit DB. Single-flight or probabilistic early expiry to reduce stampede.
- **Monitoring:** Hit rate, latency p99, node membership, memory usage.
