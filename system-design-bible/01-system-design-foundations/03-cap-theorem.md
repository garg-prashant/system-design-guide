# CAP Theorem

## The Theorem

In a distributed system, during a **network partition**, you can guarantee at most two of:

- **C**onsistency: Every read receives the most recent write or an error
- **A**vailability: Every request receives a response (not guaranteed to be the most recent)
- **P**artition tolerance: The system continues to operate despite network partitions

**The key insight:** Partition tolerance is not optional in distributed systems. Networks fail. Therefore the real choice is:

> During a partition: be **CP** (consistent but may be unavailable) or **AP** (available but may serve stale data)?

```
        Consistency
             |
             |     CP systems
             |    (Zookeeper, HBase,
             |     MongoDB w/ strict)
             |
─────────────┼─────────────
             |             \
             |              \ AP systems
             |               (Cassandra, DynamoDB,
             |                CouchDB, DNS)
             |
      Availability
```

---

## What "Partition" Means

A network partition is when nodes in a distributed system can't communicate with each other:

```
Before partition:
  Node A ←──────────────→ Node B
           network works

During partition:
  Node A ✗─────────────✗ Node B
           messages dropped

Nodes must decide: stop serving (CP) or serve potentially stale data (AP)?
```

Partitions happen because of:
- Network switch failure
- Datacenter connectivity issue
- Cloud provider network blip
- Overloaded network (packet drops)
- Misconfigured firewall

They are **not rare**. Any distributed system running in production will experience partitions.

---

## Consistency in CAP

CAP's "consistency" means **linearizability**: a system appears to operate as if there is a single copy of the data, and all operations are atomic.

This is different from the "C" in ACID transactions (which means integrity constraints are maintained).

```
Linearizability (CAP C):
  Time ──────────────────────────→

  Write X=1  ───complete──→
                              Read X  ──complete──→ must return 1

  If write completes before read starts, read MUST see the write.
```

**Not linearizable (violates CAP C):**
```
  Node A: Write X=1 completes
  Node B: (hasn't received update yet)
  Client: Read from Node B → returns X=0  ✗ violates linearizability
```

---

## AP vs CP Systems

### CP Systems

**Choose consistency over availability during partitions.**

If a node can't confirm it has the latest data (because it can't reach the other nodes), it refuses to serve requests.

Examples:
- **Zookeeper** — Leader-based, refuses reads/writes if quorum lost
- **etcd** — Raft-based, same behavior
- **HBase** — Strong consistency, may be unavailable during region server failure
- **MongoDB (w/ writeConcern: majority)** — Won't acknowledge writes until majority confirms

**When to use CP:**
- Financial systems (bank balances, payment records)
- Distributed coordination (leader election, distributed locks)
- Configuration management (wrong config → cascading failures)
- Inventory systems (prevent overselling)

### AP Systems

**Choose availability over consistency during partitions.**

Nodes serve requests even if they can't confirm they have the latest data. Systems eventually converge (eventual consistency).

Examples:
- **Cassandra** — Tunable consistency, defaults to eventual
- **DynamoDB** — Eventually consistent reads by default
- **CouchDB** — Multi-master replication with conflict resolution
- **DNS** — Updates propagate eventually; may serve stale records

**When to use AP:**
- Social media timelines (slightly stale is fine)
- Shopping carts (prefer availability over exact item count)
- User preferences/settings (minor staleness acceptable)
- Caching layers (inherently eventually consistent)

---

## Nuance: The Real World Is More Complex

### Tunable Consistency

Systems like Cassandra let you tune the consistency level per operation:

```
QUORUM reads + QUORUM writes → strong consistency (but reduced availability)
ONE reads + ONE writes → eventual consistency (maximum availability)

Quorum = (N/2) + 1 nodes must agree

For N=3:
  QUORUM = 2
  If 1 node is partitioned: QUORUM reads still work
  If 2 nodes are partitioned: QUORUM reads fail (CP behavior kicks in)
```

### Consistency Models (Stronger to Weaker)

```
Linearizability (strictest)
  → Single global ordering of all operations
  → Very expensive: 2 RTTs minimum

Sequential Consistency
  → Operations appear in some sequential order
  → Each process's operations appear in order
  → Relaxed from linearizability: no real-time guarantee

Causal Consistency
  → Causally related operations appear in order
  → Concurrent operations may appear in different orders at different nodes

Eventual Consistency (weakest useful model)
  → Given no new updates, all replicas will eventually converge
  → No timing guarantee

Read-Your-Writes Consistency (session guarantee)
  → You always see your own writes
  → Others may see stale data
```

---

## PACELC Theorem (Extension of CAP)

CAP only addresses behavior during partitions. **PACELC** also captures the normal-operation trade-off:

**During Partition (P): choose A or C**
**Else (E, normal operation): choose Latency (L) or Consistency (C)**

```
System          P → A/C    E → L/C
─────────────────────────────────────
Cassandra         AP         EL       (high availability, eventual consistency, low latency)
DynamoDB          AP         EL
Zookeeper         CP         EC       (consistent, but higher latency due to coordination)
PostgreSQL        CP         EC       (ACID = strong consistency)
MySQL (InnoDB)    CP         EC
Riak              AP         EL
```

Real-world implication: Even when the network is healthy, you're making a trade-off between latency (serve from local replica) and consistency (coordinate with all replicas).

---

## Practical CAP Decisions by Use Case

| Use Case | Recommendation | Rationale |
|----------|---------------|-----------|
| User authentication | CP | Wrong auth decision = security breach |
| Inventory/stock count | CP | Overselling has real cost |
| Bank balance reads | CP | Financial accuracy required |
| Shopping cart | AP | Better to add duplicate item than 503 |
| Social feed | AP | Slightly stale timeline is fine |
| Like count | AP | Approximate count is acceptable |
| DNS records | AP | Availability critical; staleness tolerable |
| Leader election | CP | Must have exactly one leader |
| Feature flags | AP | Brief inconsistency across users acceptable |
| Payment processing | CP | Must not double-charge |

---

## Common Misconceptions

**Misconception 1:** "We can have all three if we pick the right system."

**Reality:** During an actual partition, you must choose. There is no magic.

**Misconception 2:** "CA systems exist."

**Reality:** A CA system is just a single-node system (no partition possible). As soon as you distribute, partition tolerance is mandatory.

**Misconception 3:** "AP means the system returns wrong data."

**Reality:** AP means the system may return *stale* data, not arbitrary garbage. Eventual consistency means it will converge to the correct value.

**Misconception 4:** "CAP is the only relevant theorem."

**Reality:** CAP is a useful framework, but PACELC, consistency models, and the specific failure modes of your system matter more in practice.

---

## Architecture Diagram: CP vs AP During Partition

```
CP Behavior During Partition:
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │ Read X
                    ┌──────▼──────┐
                    │   Node A    │
                    │  Can't reach│
                    │  quorum ✗   │
                    └─────────────┘
                    Returns: 503 Service Unavailable

AP Behavior During Partition:
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │ Read X
                    ┌──────▼──────┐
                    │   Node A    │
                    │  Returns    │
                    │  local copy │
                    └─────────────┘
                    Returns: X=1 (may be stale, actual value might be X=2)
```

---

## Interview Discussion Points

- "Given this is a payment system, I'll choose CP semantics — availability is important but not at the cost of financial consistency."
- "For the social feed, I'd use an AP system — a slightly stale timeline is completely acceptable and availability is critical."
- "CAP gives us the partition-time behavior, but we should also think about PACELC: even during normal operation, do we want to pay the latency cost of strong consistency?"
- "We can use quorum reads and writes to tune the consistency level — with R + W > N, we get strong consistency while retaining some availability."
- "One mitigation for AP systems is read-your-writes consistency: users always see their own changes even if other users' changes are delayed."
