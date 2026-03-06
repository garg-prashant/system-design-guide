# System Design: Distributed Lock Service

## Problem

Design a distributed lock service: acquire and release locks (by key) across multiple clients, with timeout and optional fencing to prevent two holders from acting at the same time. High availability and correctness under failure.

## Requirements

**Functional:** Acquire lock (key, TTL/lease); release lock (key); optional renewal (heartbeat); blocking or non-blocking acquire. Prevent multiple holders for same key (safety). Optional: fencing token (monotonically increasing) for ordering.

**Non-functional:** High availability (no single point of failure); low latency; handle network partitions and process pauses (clock, GC); avoid split-brain (two holders).

## High-Level Design

```
  Client A → Lock service (e.g. Redis, etcd) → Acquire(key, TTL) → Success, return lease_id
  Client B → Acquire(key) → Fail (held by A) or wait
  Client A → Release(lease_id) or TTL expires → Lock available
```

- **Acquire:** Client requests lock for key with TTL (e.g. 10s). Server: if key not held or expired, grant lock; store (key → lease_id, expiry); return lease_id to client. Else: reject or queue (blocking).
- **Release:** Client sends release(lease_id). Server: delete key only if current holder is this lease_id (avoid releasing wrong client’s lock after pause).
- **Renewal:** Client sends heartbeat (lease_id) before TTL; server extends TTL. If client dies, no renewal → lock expires → available.
- **Fencing (optional):** Server issues monotonically increasing token with each grant; resource (e.g. storage) checks token and rejects older writes. Prevents delayed previous holder from overwriting after new holder took over.

## Key Components

- **Lock store:** Redis (SET key value NX PX ttl) or etcd (compare-and-swap with lease). Key = lock name; value = lease_id; TTL = safety (lock auto-expires).
- **Lease:** Unique per acquire; client must present lease_id to release or renew. Prevents release by wrong client after network partition.
- **Fencing token:** Monotonic counter per key; returned with lock grant; resource (e.g. DB) rejects request with token < last accepted. Mitigates “client paused then acts” problem (Redlock issue).
- **Redlock (Redis):** Acquire on N independent Redis nodes (e.g. 5); majority success = held. TTL for auto-release. Trade-off: still vulnerable to clock/ pause without fencing; use fencing on critical path.

## Data Model

- **lock:** key → (lease_id, expiry_ts, fencing_token). Stored in Redis or etcd.
- **lease_id:** UUID or unique id; returned to client; required for release and renew.

## Scale

- Lock operations are relatively low volume (coordination, not per-request). Redis or etcd cluster for HA; low latency (single-digit ms).

## Trade-offs

- **Redis vs etcd:** Redis: simple, Redlock for multi-node. etcd: consensus-based, lease and compare-and-swap native; stronger consistency. Use etcd (or similar) when correctness is critical.
- **TTL:** Short TTL = quick recovery if holder dies; more renewal traffic. Long TTL = less renewal; longer wait if holder dies without release.
- **Fencing:** Adds complexity (resource must check token); required for critical resources (e.g. storage) to avoid split-brain. Simple lock without fencing is OK for non-critical coordination.

## Failure & Operations

- **Holder dies:** TTL expires; lock released; another client can acquire. Ensure TTL is longer than max GC pause or clock skew if possible.
- **Network partition:** Holder may think it holds lock but server expired it; new holder granted. Fencing token prevents old holder from acting on resource.
- **Monitoring:** Lock acquire/release latency, contention (wait time), expiry rate, fencing token usage.
