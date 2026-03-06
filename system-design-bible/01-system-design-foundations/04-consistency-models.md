# Consistency Models

## Why Consistency Models Matter

In a distributed system with replicated data, different clients may see different versions of the same data at the same time. Consistency models define the **contract** between the system and its users about what they can observe.

Choosing the right consistency model is one of the most impactful architectural decisions because it directly affects:
- System latency (stronger = slower)
- Availability during failures
- Complexity of application logic
- Data correctness guarantees

---

## Consistency Spectrum (Strongest to Weakest)

```
Strict Serializability
       │    (linearizability + serializability)
       ↓
Linearizability (atomic consistency)
       │
       ↓
Sequential Consistency
       │
       ↓
Causal+ Consistency
       │
       ↓
Causal Consistency
       │
       ↓
FIFO / PRAM Consistency
       │
       ↓
Read-Your-Writes
       │
       ↓
Monotonic Reads
       │
       ↓
Eventual Consistency (weakest useful model)
```

---

## Linearizability

**Definition:** The system behaves as if there is a single copy of the data, and all operations take effect atomically at some point between their start and completion.

**Key property:** Real-time ordering is preserved. If operation A completes before operation B starts, B must see A's effects.

```
Time ─────────────────────────────────────────→

Client 1:  [Write X=1 ────────complete]
Client 2:                                [Read X] → must return 1

Client 3:  [Write X=2 ──────complete]
Client 4:              [Read X] → may return 1 or 2 (concurrent with write)
           But once client 4 reads X=2, all subsequent reads must see ≥ 2
```

**Implementation:** Requires coordination (Paxos, Raft) — every write must be confirmed by a quorum before returning.

**Cost:** 2+ RTTs for every write; significant latency overhead.

**Examples:** etcd, Zookeeper, CockroachDB, Spanner (using TrueTime).

**Use when:** Distributed locks, leader election, inventory counters, banking.

---

## Sequential Consistency

**Definition:** All operations appear to execute in some sequential order, and each individual process's operations appear in the order specified by the program.

**Difference from linearizability:** No real-time constraint. The global ordering doesn't need to match wall-clock time.

```
Process 1: Write X=1, Write Y=2
Process 2: Read Y=2, Read X=0   ← valid in sequential consistency
                                   (X=0 because process 2's reads were ordered
                                    before process 1's writes in some valid sequence)

This would be INVALID in linearizability if P1 completed before P2 started reading.
```

**Examples:** Java memory model (without volatile), some GPU memory models.

---

## Causal Consistency

**Definition:** Operations that are causally related (one happens-before another) are seen in the same order by all nodes. Concurrent operations may be seen in different orders.

**Happens-before:** A → B if:
- A and B are by the same process, and A occurred before B
- A is a send event and B is the corresponding receive
- A → C and C → B (transitivity)

```
Process 1: Write X=1 ────→ [message] ────→ Process 2: Read X (gets 1)
                                               │
                                               └──→ Write Y=2

Process 3: Read Y (might get 0 or 2, but if it reads 2, it must also see X=1)
```

**Why it matters:** Many real applications only care about causal ordering, not total ordering. Causal consistency can be implemented with much lower latency than linearizability.

**Implementation:** Vector clocks or logical timestamps track causality.

**Examples:** MongoDB (sessions), some Cassandra configurations, Bayou.

---

## Eventual Consistency

**Definition:** If no new updates are made to an object, eventually all accesses will return the last updated value.

**What it does NOT guarantee:**
- Timing of convergence
- What value is returned before convergence
- Order in which updates are seen

**What it DOES guarantee:**
- Given enough time with no new writes, all replicas converge

```
Time:  t0     t1     t2     t3     t4
Node A: X=1   X=1   X=2   X=2   X=2
Node B: X=1   X=1   X=1   X=2   X=2
Node C: X=1   X=1   X=1   X=1   X=2
                                  ↑
                            All nodes agree = eventual consistency achieved
```

**Practical implications:**
1. Applications must handle stale reads
2. Write conflicts must be resolved (LWW, CRDT, user-defined)
3. "Eventually" can mean milliseconds or hours depending on network conditions

**Examples:** DNS, Cassandra (default), DynamoDB (default reads), S3 (historically), social media feeds.

---

## Read-Your-Writes (Session Consistency)

**Definition:** Within a session, a client always sees the effects of its own writes.

Reads that follow a write in the same session will see the write, even if they go to a different replica.

```
Client session:
  Write X=5  ──→ Primary: X=5
  Read X  ──→ Replica (may be stale) but returns X=5 because session sticky
```

**Implementation:**
1. Sticky sessions (route session to same server)
2. Read from primary after writes
3. Include write timestamp in subsequent reads; replica returns result only if it's up-to-date

**Examples:** Most SQL databases for single sessions, Cassandra with SERIAL/QUORUM, DynamoDB with strongly consistent reads.

---

## Monotonic Reads

**Definition:** If a process reads X and sees value V, subsequent reads of X will never return a value older than V.

```
Valid:
  Read X → 1
  Read X → 1  (same or newer)
  Read X → 2  (newer, fine)

Invalid:
  Read X → 2
  Read X → 1  ← went backward in time!
```

**Implementation:** Route a client to the same replica (sticky), or track vector clock and ensure replica is at least as up-to-date.

---

## Monotonic Writes

**Definition:** A process's writes are applied in the order they were issued.

```
Process:
  Write X=1  (must be applied before)
  Write X=2

All replicas must eventually apply these in order, not X=2 then X=1.
```

---

## Conflict Resolution in Eventual Consistency

When multiple clients write to the same key concurrently in an AP system, conflicts arise. Common resolution strategies:

### Last Write Wins (LWW)

Use timestamps; highest timestamp wins.

```
Write A: X=1, timestamp=100
Write B: X=2, timestamp=105

Result: X=2 (higher timestamp wins)
```

**Problem:** Clock skew between nodes means timestamps can be unreliable. A later write with a lower clock value gets discarded.

### Multi-Value / Siblings

Keep all conflicting values, return to client for resolution.

```
Conflicting writes detected:
  Version 1: X=1
  Version 2: X=2

Return both; client or application logic decides which to keep.
```

**Example:** Dynamo-style shopping carts — union of items.

### CRDTs (Conflict-Free Replicated Data Types)

Design data structures that can be merged deterministically without conflicts.

```
Counter CRDT:
  Node A: +3
  Node B: +2
  Merge: max(A_count, B_count) = 5 (eventually correct regardless of order)

Set CRDT (G-Set, grow-only):
  Node A: {a, b}
  Node B: {b, c}
  Merge: {a, b, c}
```

**Tradeoff:** Only some data structures can be expressed as CRDTs. Deletions are complex.

### Vector Clocks

Track causality to detect and resolve conflicts:

```
Event sequence:
  A writes X=1 at clock [A:1, B:0]
  B reads X, sees X=1 at clock [A:1, B:0]
  B writes X=2 at clock [A:1, B:1]  (causally descended from A's write)

  A writes X=3 at clock [A:2, B:0]  (concurrent with B's write!)

  Both [A:2, B:0] and [A:1, B:1] are incomparable → conflict!
  Need resolution strategy (LWW, user merge, etc.)
```

---

## Consistency in Databases

### ACID Transactions (OLTP)

| Property | Meaning |
|----------|---------|
| Atomicity | All or nothing — partial writes don't persist |
| Consistency | Database invariants are maintained |
| Isolation | Concurrent transactions don't interfere |
| Durability | Committed data survives crashes |

**Isolation levels (SQL standard):**
```
Read Uncommitted  (weakest, fastest)
  └── Dirty reads possible

Read Committed  (PostgreSQL default)
  └── No dirty reads; non-repeatable reads possible

Repeatable Read  (MySQL InnoDB default)
  └── Snapshot of data at transaction start

Serializable  (strongest, slowest)
  └── Transactions appear to run one at a time
```

**Common anomalies by isolation level:**

| Anomaly | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|---------|:----------------:|:--------------:|:---------------:|:------------:|
| Dirty Read | ✗ | ✓ | ✓ | ✓ |
| Non-repeatable Read | ✗ | ✗ | ✓ | ✓ |
| Phantom Read | ✗ | ✗ | ✗ | ✓ |
| Serialization anomaly | ✗ | ✗ | ✗ | ✓ |

(✓ = protected, ✗ = vulnerable)

### BASE (NoSQL alternative to ACID)

- **B**asically Available
- **S**oft state (state may change over time without input)
- **E**ventually consistent

BASE trades correctness guarantees for availability and performance.

---

## Choosing a Consistency Model

```
Need correctness above all?
  Yes → Linearizability (etcd, Zookeeper, Spanner)

Need transactional correctness?
  Yes → Serializable ACID (PostgreSQL, CockroachDB)

Can tolerate slight staleness, need low latency?
  Yes → Read-your-writes + eventual consistency (Cassandra, DynamoDB)

High write throughput, can merge conflicts?
  Yes → CRDTs + eventual consistency

Simple key-value, last write wins is fine?
  Yes → LWW + eventual consistency (many caches, Redis)
```

---

## Interview Discussion Points

- "For this feature, read-your-writes consistency is sufficient — users need to see their own edits immediately, but don't need to see other users' changes in real-time."
- "We're using Cassandra here with QUORUM reads and writes, which gives us strong consistency with N=3 replicas. We're sacrificing some availability but getting the correctness we need."
- "Conflict resolution strategy matters: for this shopping cart, I'd prefer multi-value (union) over LWW to avoid losing items due to clock skew."
- "Using vector clocks adds complexity to the client; I'd only recommend CRDTs if the access pattern naturally fits the data structure constraints."
- "Eventually consistent systems need idempotent write operations — if we retry on failure, we might apply the same write twice."
