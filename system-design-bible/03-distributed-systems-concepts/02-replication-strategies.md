# Replication Strategies

## Why Replicate?

1. **Availability** — If one copy fails, others serve requests
2. **Durability** — Data not lost when a node fails
3. **Performance** — Serve reads from geographically closer replicas
4. **Scalability** — Distribute read load across replicas

---

## Single-Leader Replication (Master-Slave)

The most common replication model.

```
        ┌──────────────┐
Writes → │    Primary   │
         │   (Leader)   │
        └──────┬───────┘
               │ replicate
    ┌──────────┼──────────┐
    ↓          ↓          ↓
┌────────┐ ┌────────┐ ┌────────┐
│Replica │ │Replica │ │Replica │
│   1    │ │   2    │ │   3    │
└────────┘ └────────┘ └────────┘
    ↑          ↑          ↑
           Reads
```

**Write path:** All writes go to the primary. Primary writes to its log, then sends the changes to replicas.

**Read path:** Reads can go to any replica (eventual consistency) or primary (strong consistency).

### Synchronous vs Asynchronous Replication

**Synchronous:**
```
Client → Write → Primary: apply, send to replicas
         Wait ←─────────────── Replica 1: ack
                     (Replica 2 may be async)
Client ← Response (data is durable on primary + Replica 1)
```
- Primary waits for at least one replica to confirm before acknowledging client
- **Pro:** No data loss if primary fails (replica has the data)
- **Con:** Write latency increases; if replica is slow, primary is slow
- **Con:** If synchronous replica fails, writes block (or temporarily go async)

**Asynchronous:**
```
Client → Write → Primary: apply, return success
Client ← Response
         Primary → Replicas: replicate (in background)
```
- Primary returns success without waiting for replicas
- **Pro:** Lower write latency
- **Con:** Failover can lose data (replicas may not have latest writes)
- **Pro:** Primary can proceed even if replicas are down

**Semi-synchronous (common production approach):**
- One replica is synchronous, others are asynchronous
- Guarantees at least 2 copies on failover
- Balance between durability and performance

### Replication Lag

The delay between a write on the primary and visibility on replicas.

**Causes:**
- Network latency (same DC: ~1ms; cross-region: ~50-150ms)
- Replica under load (processing previous writes)
- Large transactions (must replay entire transaction)

**Problems caused by replication lag:**

1. **Read-after-write inconsistency:**
   ```
   User updates profile → writes to primary
   User refreshes page → reads from replica (stale!)
   ```
   Fix: Route user's own reads to primary for a short window, or track write timestamp.

2. **Monotonic read violation:**
   ```
   Read from Replica 1: gets value 5
   Read from Replica 2: gets value 3 (more lagged)
   User sees time going backward!
   ```
   Fix: Sticky routing per session.

3. **Causality violation:**
   ```
   User A posts comment
   User B reads post (from ahead replica)
   User B reads comments (from behind replica) — comment not visible yet!
   ```
   Fix: Use causal consistency tokens.

### Leader Failover

When the primary fails:
1. Detect failure (timeout-based; typically 10-30 seconds)
2. Elect new leader (choose replica with most up-to-date data)
3. Redirect clients to new leader
4. Old leader (if it comes back) must become follower

**Split-brain risk:** Old leader comes back and thinks it's still leader. Both leaders accept writes → divergence.
**Solution:** Fencing token (old leader gets higher epoch, reject old leader's writes); STONITH (shoot the other node in the head).

---

## Multi-Leader Replication

Multiple nodes accept writes. Writes asynchronously replicated to all other leaders.

```
  Region A          Region B
  ┌────────┐        ┌────────┐
  │Leader A│ ←────→ │Leader B│
  │        │        │        │
  └────────┘        └────────┘
  Accepts            Accepts
  writes             writes
```

**Use cases:**
- Multi-datacenter deployments (avoid cross-DC write latency)
- Offline operation (each device is a "leader" while offline; syncs later)
- Collaborative editing (each client is a leader)

**The fundamental problem — write conflicts:**

If Leader A and Leader B both write to the same key concurrently, what's the final value?

```
Time   Leader A        Leader B
t1:    X = 1
t2:                    X = 2
t3:    replicate X=1 → B
t4:    ← replicate X=2

Both leaders see a conflict: X is 1 or 2?
```

**Conflict resolution strategies:**

1. **Avoid conflicts:** Route all writes for a given record to the same leader (e.g., by user ID)
2. **Last Write Wins (LWW):** Use timestamp; highest wins. Risk: clock skew causes data loss.
3. **Application-level merge:** Present both versions; application or user resolves (e.g., Google Docs)
4. **CRDT:** Use data structures that automatically merge (see `distributed-systems/crdts.md`)

---

## Leaderless Replication (Dynamo-style)

No leader. Any node can accept reads and writes. Clients send requests to multiple nodes simultaneously.

```
         ┌────────────┐
         │   Client   │
         └──┬──┬──┬───┘
            │  │  │
     ┌──────┘  │  └──────┐
     ↓         ↓         ↓
  ┌──────┐  ┌──────┐  ┌──────┐
  │Node 1│  │Node 2│  │Node 3│
  └──────┘  └──────┘  └──────┘
```

**Write:** Client sends to W nodes simultaneously. Success when W nodes respond.

**Read:** Client sends to R nodes simultaneously. Takes the most recent value (determined by version number/timestamp). Success when R nodes respond.

### Quorum Reads and Writes

With N replicas:
- Write quorum: W (must acknowledge write)
- Read quorum: R (must respond to read)

**For strong consistency:** R + W > N

Example: N=3, W=2, R=2 → R+W=4>3 → consistent

```
Why R+W>N ensures consistency:
  W=2 means write is on at least 2 of 3 nodes
  R=2 means we read from at least 2 of 3 nodes
  Overlap: at least 1 node is in both W and R sets
  That node has the latest write → we get latest value
```

**Tunable consistency:**
- W=N, R=1: Maximum write durability, fast reads
- W=1, R=N: Fast writes, maximum read consistency
- W=1, R=1: Maximum performance, minimal durability
- W=2, R=2 (N=3): Balance (most common)

### Sloppy Quorum and Hinted Handoff

During a partition, some preferred nodes may be unreachable. Rather than returning error, write to other available nodes ("sloppy quorum"), tagging data with original destination ("hint"). When original node recovers, replay the hinted writes.

**Effect:** Higher availability, but temporarily may not meet W quorum on original nodes.

---

## Replication Topologies

### Circular Replication
```
A → B → C → A
```
One broken node can interrupt the chain.

### Star Replication
```
    B
    ↑
A → Primary → C
    ↓
    D
```
Primary is bottleneck; failure stops all replication.

### Full Mesh
```
A ↔ B ↔ C
↕       ↕
D ↔─────┘
```
Every node replicates to every other. O(n²) connections. Used for small N (3-5 nodes).

---

## Replication in Practice

### PostgreSQL Streaming Replication

```
Primary:
  WAL (Write-Ahead Log) stream → Replica: applies WAL entries

Replica lag monitoring:
  SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
  AS replication_lag_seconds;
```

**Synchronous replication config:**
```sql
-- On primary:
synchronous_standby_names = 'replica1'
synchronous_commit = on  -- wait for replica before ACK

-- Writes wait for replica1 to confirm receipt
-- But: if replica1 is down, writes block!
```

### MySQL Group Replication

- Multi-primary mode: any node accepts writes
- Paxos-based consensus for conflict detection
- Automatically detects and rejects conflicting transactions

### Kafka Replication

```
Topic partition: 3 replicas
  Leader replica: handles reads and writes
  Follower replicas: replicate from leader

ISR (In-Sync Replicas): followers that are up-to-date
  Write is committed only when all ISR replicas confirm
  Leader failure: new leader elected from ISR
```

---

## Replication Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Primary crashes | Writes blocked until failover | Auto-failover, fast detection |
| Replica crashes | Reduced read capacity; sync replica → primary blocked | Remove from LB, repair replica |
| Replication lag grows | Stale reads; failover loses data | Monitor lag, alert on threshold |
| Split-brain | Two primaries accept conflicting writes | Fencing tokens, STONITH |
| Network partition | Replicas can't receive writes | Quorum-based acceptance |
| Corrupted replication stream | Replica gets out of sync | Checksums, logical replication |

---

## Cross-Region Replication

For global systems:

```
us-east-1 (primary)       eu-west-1 (replica)       ap-southeast-1 (replica)
┌─────────────┐           ┌─────────────┐           ┌─────────────┐
│  Primary DB │ ─────────→│  Replica DB │           │  Replica DB │
│             │ ←──────── │             │           │             │
└─────────────┘           └─────────────┘           └─────────────┘
     ~100ms RTT                                    ~200ms RTT
```

**Challenges:**
- Cross-region latency (50-200ms) → replication lag is high
- Strong consistency across regions is extremely expensive (writes must cross ocean)
- Conflicts if multi-leader across regions

**Common patterns:**
1. **Single primary, async cross-region replicas:** Accept replication lag; promote regional replica on region failure
2. **Active-active with conflict resolution:** Higher complexity; CRDTs or LWW
3. **Route by geography:** Users in EU write to EU primary; replicate to other regions

---

## Interview Discussion Points

- "Replication factor of 3 gives us N+1 redundancy and quorum-based consistency. For critical data, I wouldn't go below 3."
- "Asynchronous replication means we could lose up to the lag window of data on primary failure. For this payment system, I'd use synchronous replication to at least one replica."
- "We need to monitor replication lag and alert if it exceeds our SLO threshold. Growing lag is an early warning sign of a replica under-provisioned or a replication bottleneck."
- "Read-after-write consistency: after a user writes, we route their subsequent reads to the primary (or add a version token to their read requests) for 30 seconds."
- "Multi-leader replication adds significant complexity in conflict resolution. I'd only use it for multi-region writes where latency is the bottleneck — otherwise single-leader with a well-placed primary is simpler."
