# Consensus Algorithms

## The Problem

In a distributed system, multiple nodes must agree on a single value despite failures and unreliable networks. This is the **consensus problem**.

**Why it's hard:**
- Nodes can fail (crash, Byzantine)
- Messages can be delayed, duplicated, or lost
- No global clock for ordering events
- Network partitions can split nodes into subgroups

**Formal requirements:**
1. **Agreement:** All non-faulty nodes decide on the same value
2. **Validity:** The decided value was proposed by some node (not invented)
3. **Termination:** All non-faulty nodes eventually decide

**FLP Impossibility:** In a purely asynchronous system, consensus is impossible with even one faulty process. This is why real systems use timeouts (introducing synchrony assumptions).

---

## Use Cases for Consensus

- Leader election (one node acts as coordinator)
- Distributed locking (one process holds lock)
- Atomic broadcast (all nodes receive same messages in same order)
- Replicated state machine (all replicas apply same operations in same order)
- Distributed transactions (all-or-nothing across nodes)

---

## Paxos

The foundational consensus algorithm, proposed by Leslie Lamport (1989).

### Roles

- **Proposer:** Initiates proposals (client or dedicated node)
- **Acceptor:** Votes on proposals (quorum of acceptors form the decision)
- **Learner:** Learns the decided value (may overlap with acceptors)

### Two-Phase Protocol

```
Phase 1: Prepare

Proposer chooses proposal number N (higher than any seen before)
Proposer → Acceptors: Prepare(N)

Acceptor response:
  If N > any previous prepare it has responded to:
    Promise not to accept proposals numbered < N
    Return (if any) the highest-numbered proposal it has accepted
  Else:
    Ignore or Nack

Proposer needs majority (quorum) of acceptors to respond with promises.
```

```
Phase 2: Accept

Proposer picks value V:
  - If any acceptor returned an accepted value, use the highest-numbered one
  - Otherwise, proposer can use its own value

Proposer → Acceptors: Accept(N, V)

Acceptor response:
  If no promise has been made to a proposal numbered > N:
    Accept (N, V)
    Notify learners
  Else:
    Ignore (a higher-numbered proposal is in flight)

When a quorum of acceptors accept (N, V), V is chosen.
```

### Why Quorum?

With N acceptors, we need a quorum of (N/2 + 1). Any two quorums overlap by at least one node. This overlap ensures the chosen value is "remembered" even if some acceptors fail.

```
N=5 acceptors, quorum=3
  Round 1: Acceptors {A, B, C} accept value V1
  Round 2: Proposer 2 gets promises from {C, D, E}
           C is in both quorums → C tells proposer 2 about V1
           Proposer 2 must propose V1, not a new value
```

### Paxos Limitations

- **Single value:** Basic Paxos decides one value. Multi-Paxos chains instances to decide a sequence (log).
- **Liveness issue:** Two competing proposers can keep out-bidding each other (livelock). Need randomized backoff or leader election.
- **Complexity:** Hard to implement correctly.

---

## Raft

Designed to be more understandable than Paxos while providing equivalent guarantees.

### Key Concepts

- **Leader:** Single node that handles all writes; replicates to followers
- **Term:** Monotonically increasing logical time unit; each term has at most one leader
- **Log:** Ordered sequence of commands; all nodes must agree on the log
- **Commit:** An entry is committed when replicated to a majority

### Leader Election

```
State machine for each node:
  Follower → Candidate → Leader

  Follower:
    - Receives heartbeats from leader
    - If election timeout expires with no heartbeat → become Candidate

  Candidate:
    - Increments term, votes for self
    - Sends RequestVote RPCs to all nodes
    - Wins if majority votes received → become Leader
    - If another leader responds → revert to Follower
    - If election timeout expires → start new election

  Leader:
    - Sends AppendEntries RPCs (heartbeats + log replication)
    - Handles client requests
    - Resigns if it discovers a higher term
```

```
Term Timeline:
  Term 1: Leader A elected, leads
  ──────────────────────────────────→

  (Network partition, A isolated)

  Term 2: B becomes candidate, wins election, leads
  ──────────────────────────────────────────────────→

  (A rejoins)
  A sees term 2 > term 1, reverts to follower
```

### Log Replication

```
  Client: Write X=1

  Leader: Appends (term=2, index=5, X=1) to local log
  Leader → Follower B: AppendEntries(prev_index=4, entry=(2,5,X=1))
  Leader → Follower C: AppendEntries(prev_index=4, entry=(2,5,X=1))

  Followers:
    - Verify prev_index matches (consistency check)
    - Append to log
    - Send success response

  Leader:
    - Once majority (including itself) have appended: entry is committed
    - Apply to state machine, return success to client
    - Notify followers of commit in next heartbeat
```

### Safety Properties

1. **Election Safety:** At most one leader per term
2. **Leader Append-Only:** A leader never overwrites its log
3. **Log Matching:** If two logs have an entry with same index and term, all preceding entries are identical
4. **Leader Completeness:** If an entry is committed, all future leaders have it
5. **State Machine Safety:** If a server applies entry at index I, no other server applies a different entry at index I

### Raft Configuration Changes

Adding/removing nodes requires careful handling to avoid split-brain during transition. Raft uses joint consensus: a two-phase transition period where both old and new configurations require quorum.

---

## Zab (Zookeeper Atomic Broadcast)

Used internally by Apache Zookeeper. Similar to Paxos/Raft but optimized for:
- Delivering changes in order
- Leader recovery preserving all previously broadcast messages

Difference from Raft: Zab separates the discovery/synchronization phases and is designed specifically for primary-backup systems rather than general state machines.

---

## Viewstamped Replication (VSR)

Pre-dates Paxos; less well-known but equivalent. Basis for some distributed database systems.

---

## Byzantine Fault Tolerance

The above algorithms handle **crash failures** (nodes stop responding). **Byzantine fault tolerance** (BFT) handles nodes that actively lie or send inconsistent messages (hardware errors, compromised nodes, bugs).

**PBFT (Practical Byzantine Fault Tolerance):**
- Handles up to f Byzantine nodes with 3f+1 total nodes
- Three phases: pre-prepare, prepare, commit
- O(n²) message complexity — expensive for large clusters
- Used in blockchain systems, some financial systems

**Why typically not used:**
- Very high overhead (requires 3f+1 nodes vs 2f+1 for crash-only)
- Complex implementation
- Most production systems don't face Byzantine failures
- Use for: blockchain consensus (Tendermint, PBFT), systems with untrusted nodes

---

## Practical Consensus Systems

| System | Algorithm | Used By | Notes |
|--------|-----------|---------|-------|
| etcd | Raft | Kubernetes, many distributed systems | Production-grade, well-tested |
| Zookeeper | Zab | Kafka (older), HBase, Hadoop | Very mature, more complex API |
| Consul | Raft | Service discovery, KV store | From HashiCorp |
| CockroachDB | Raft (per-range) | Distributed SQL | Each shard runs Raft independently |
| TiKV | Raft (multi-raft) | TiDB | Inspired by CockroachDB |
| Spanner | Paxos | Google Cloud Spanner | Combines with TrueTime for external consistency |
| MongoDB (replica sets) | Raft-like | MongoDB | Primary-secondary replication |

---

## Performance Characteristics

```
Paxos/Raft single-leader write:
  Client → Leader → (Replicate to F) → Commit → Response

  Minimum latency: 1 RTT (leader → followers) + processing
  With geo-distribution: ~100ms per RTT

  Throughput: Limited by leader's network and disk I/O
  Typical: 10K-100K operations/sec per Raft group

Multi-Raft (sharded):
  Horizontal scaling: N Raft groups → N × throughput
  Used by: CockroachDB, TiKV, Spanner
```

---

## Common Interview Questions

**Q: How does Raft handle a split-brain scenario?**

A: During a partition, the majority partition can still elect a leader and make progress (quorum = majority). The minority partition can't elect a leader (can't get majority votes), so it stalls. When the partition heals, the minority nodes adopt the leader's log. No two partitions can both have a quorum, so no split-brain.

**Q: What happens to in-flight writes during a leader failure in Raft?**

A: Writes that hadn't been committed (replicated to majority) are lost — they'll be re-sent by the client (if it retries). Writes that were committed are safe: the new leader must have them in its log (because it got a majority vote, and committed entries are on a majority).

**Q: Why does Raft require log entries to be committed before applying to state machine?**

A: A leader might commit entry E, but if it crashes before telling followers it's committed, the entry is still in the logs of a majority. A new leader will be elected from one of those majority nodes, and will re-commit E. Applying entries only after commit ensures all nodes apply the same final sequence.

---

## Interview Discussion Points

- "For leader election in this service, I'd use etcd rather than implementing Raft from scratch — it's a solved problem and etcd is battle-tested."
- "We're using a Raft-based system, so our write latency is at minimum one RTT to replicas. For cross-region, that's 100ms+. We should factor this into our latency budget."
- "The quorum requirement (N/2+1) means we can tolerate floor(N/2) failures. With 3 nodes, we tolerate 1 failure. With 5 nodes, 2 failures. I'd recommend 5 nodes for a critical coordination service."
- "Consensus is expensive — we shouldn't use it for every operation. Use it for metadata (who is leader, what's the config), and use leader-based replication for data."
