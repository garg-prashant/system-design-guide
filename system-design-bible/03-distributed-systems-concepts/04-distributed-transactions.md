# Distributed Transactions

## The Problem

In a monolith with a single database, ACID transactions handle consistency automatically. In a distributed system with multiple services and databases, maintaining consistency across service boundaries is hard.

**The core challenge:**

```
Order Service (DB: orders)     Payment Service (DB: payments)
  Create order                   Charge customer

What if one succeeds and the other fails?
  - Order created, payment failed: order exists with no payment
  - Payment charged, order not created: money taken, no order
```

---

## Two-Phase Commit (2PC)

The classic protocol for distributed atomicity.

### Roles

- **Coordinator:** Orchestrates the transaction (often the initiating service)
- **Participants:** Services or databases involved in the transaction

### Protocol

```
Phase 1: Prepare (Voting)

  Coordinator → Participant A: "Prepare to commit"
  Coordinator → Participant B: "Prepare to commit"

  Participant A: Executes transaction, acquires locks, writes to redo log
  Participant A → Coordinator: "PREPARED" (or "ABORT")

  Participant B: Same
  Participant B → Coordinator: "PREPARED" (or "ABORT")
```

```
Phase 2: Commit (or Abort)

  If ALL participants voted PREPARED:
    Coordinator → all: "COMMIT"
    Participants: apply transaction, release locks

  If ANY participant voted ABORT or timeout:
    Coordinator → all: "ABORT"
    Participants: roll back, release locks
```

### Failure Cases

```
Failure 1: Participant crashes after PREPARED but before COMMIT received
  - Participant restarts, sees it was PREPARED
  - Queries coordinator: "Did we commit?"
  - Coordinator responds: COMMIT or ABORT
  - Participant acts accordingly

Failure 2: Coordinator crashes after sending PREPARE but before COMMIT
  - Participants are in "prepared" state, holding locks
  - BLOCKED — cannot proceed without coordinator!
  - This is 2PC's fatal flaw: blocking protocol

Failure 3: Coordinator crashes after sending COMMIT to some but not all
  - Some participants commit, others don't
  - Inconsistency!
  - Recovery: use coordinator's log to determine intent, replay
```

### 2PC Drawbacks

1. **Blocking:** If coordinator fails after prepare phase, participants hold locks indefinitely until coordinator recovers
2. **Latency:** 2 round trips minimum; more with failures
3. **Availability:** If any participant is unavailable, transaction cannot proceed
4. **Not scalable:** Coordinator is a bottleneck; lock holding reduces throughput

### When to Use 2PC

- Within a single organization's infrastructure (not across external APIs)
- When atomic cross-service transactions are truly required
- Database systems: PostgreSQL, MySQL, Oracle support XA (standardized 2PC)
- Not for high-throughput or high-availability requirements

---

## Three-Phase Commit (3PC)

Adds a "pre-commit" phase to make the protocol non-blocking.

```
Phase 1: CanCommit (voting)
Phase 2: PreCommit (coordinator has decided, participants can safely commit)
Phase 3: DoCommit (finalize)
```

**Advantage:** Non-blocking — participants can unilaterally decide based on phase 2 state.
**Disadvantages:** More messages; still vulnerable to network partitions; rarely used in practice.

---

## Saga Pattern

**Alternative to 2PC for microservices.** Break a long-running transaction into a sequence of local transactions, each with a compensating transaction for rollback.

```
Long transaction: Create order, reserve inventory, charge payment

Saga:
  T1: Create order
  T2: Reserve inventory
  T3: Charge payment

If T3 fails:
  C2: Cancel inventory reservation (compensating transaction for T2)
  C1: Cancel order (compensating transaction for T1)
```

### Choreography-Based Saga

Services react to events published by other services. No central coordinator.

```
OrderService:
  Create order
  Publish: "order.created" event

InventoryService:
  Listen: "order.created"
  Reserve inventory
  Publish: "inventory.reserved" OR "inventory.failed"

PaymentService:
  Listen: "inventory.reserved"
  Charge payment
  Publish: "payment.completed" OR "payment.failed"

OrderService:
  Listen: "payment.completed" → mark order confirmed
  Listen: "payment.failed" → cancel order, publish "order.cancelled"

InventoryService:
  Listen: "order.cancelled" → release inventory reservation
```

**Pros:**
- No single point of failure (no coordinator)
- Loose coupling

**Cons:**
- Hard to understand/track overall saga state
- Difficult to debug (events scattered across services)
- Cyclic dependencies can form

### Orchestration-Based Saga

A central coordinator (orchestrator) directs each participant.

```
┌─────────────────────────────────────────┐
│            Saga Orchestrator            │
│                                         │
│  Step 1: Call OrderService.createOrder  │
│  Step 2: Call InventoryService.reserve  │
│  Step 3: Call PaymentService.charge     │
│                                         │
│  On failure at step N:                  │
│    Call compensation for steps N-1, N-2, ... │
└─────────────────────────────────────────┘
```

**Pros:**
- Clear flow — easy to understand and debug
- Centralized state tracking
- Easier to add new steps

**Cons:**
- Orchestrator is a central point of complexity (not failure — it can be replicated)
- Tight coupling to orchestrator logic
- Risk of orchestrator becoming a "God" service

### Compensating Transactions

Must be designed carefully:

**Idempotent:** Compensation might run multiple times (on retry). Must be safe to apply multiple times.

**Not always possible:** Some actions can't be undone (sent email, shipped package). Use "semantic rollback" — cancel the business effect, not the technical action. (Send cancellation email, issue refund, etc.)

**Compensations must be retried until success:** A failing compensation means partial rollback. Must have durable retry mechanism (workflow engine, outbox pattern).

---

## Outbox Pattern

Ensures atomic "write to DB + publish event" without distributed transaction.

```
Problem:
  Write to DB: success
  Publish to Kafka: network failure!
  Event never delivered → inconsistency

Outbox solution:
  Same DB transaction:
    1. Write main record to DB
    2. Write event to "outbox" table (same transaction, atomically)

  Separate relay process:
    3. Poll outbox table for unpublished events
    4. Publish to message broker
    5. Mark as published (or delete)
```

```sql
-- orders table + outbox in same transaction
BEGIN;
  INSERT INTO orders (id, user_id, status) VALUES (123, 456, 'pending');
  INSERT INTO outbox (event_type, payload) VALUES (
    'order.created',
    '{"order_id": 123, "user_id": 456}'
  );
COMMIT;

-- Relay reads outbox and publishes
-- Exactly-once delivery to broker via idempotent producer + dedup on consumer
```

**Relay implementation options:**
1. **Polling relay:** Simple; SELECT * FROM outbox WHERE published=false
2. **CDC (Change Data Capture):** Tail the DB WAL (Debezium); real-time, no polling overhead

**Benefits:** Atomicity guaranteed by single-DB transaction. No 2PC. Publisher failure is safe (outbox entry persists until successfully relayed).

---

## Transactional Inbox (Idempotent Consumer)

The counterpart to the outbox: ensure that consuming an event is idempotent and exactly once.

```
Consumer reads event from broker
→ Process event
→ Write result to DB

Problem: Consumer crashes after processing but before committing offset.
Broker redelivers event → duplicate processing!

Solution: Inbox table
  Same DB transaction:
    1. INSERT INTO inbox (event_id) VALUES (:event_id)  ← unique constraint
    2. Apply business logic
    3. Write results
  COMMIT;

If event_id already in inbox: transaction fails → skip (idempotent)
```

---

## Long-Running Transactions (Process Manager / Workflow)

For complex multi-step processes spanning minutes to days:

```
Order fulfillment workflow:
  Step 1: Create order
  Step 2: Reserve inventory
  Step 3: Process payment
  Step 4: Allocate warehouse
  Step 5: Ship
  Step 6: Deliver

  Each step may take minutes to hours
  External systems involved
  Must handle failures, retries, timeouts, human approvals
```

**Solutions:**
- **Temporal:** Workflow-as-code; handles failures, retries, state persistence
- **AWS Step Functions:** State machine-based workflow orchestration
- **Airflow / Prefect:** For data pipeline workflows
- **Custom workflow table:** Store workflow state in DB; cron or queue drives progress

---

## Idempotency

Critical for distributed transactions and retries.

**Idempotent:** Applying the same operation multiple times produces the same result as applying once.

```
Idempotent:
  PUT /users/123 {name: "Alice"}  → always sets name to Alice
  DELETE /users/123               → user is deleted (subsequent deletes are no-ops)

Not idempotent:
  POST /orders          → each call creates a new order!
  POST /account/credit  → each call adds money!
```

**Making non-idempotent operations idempotent:**

1. **Idempotency key:** Client generates unique key per operation. Server stores key → result mapping.

```
POST /orders
Headers: Idempotency-Key: client-uuid-abc123

Server:
  Check if idempotency-key exists in DB
  If yes: return previous result (don't re-execute)
  If no:  execute, store (idempotency-key → result), return result
```

2. **Natural idempotency:** Use PUT instead of POST where possible. Store the intent, not the action.

3. **Deduplication by content hash:** Hash the request payload; reject duplicates within time window.

---

## Exactly-Once Semantics

**At-most-once:** Message may be lost, never duplicated. (Fire and forget)
**At-least-once:** Message guaranteed delivered, but may be duplicated. (Most common)
**Exactly-once:** Message delivered exactly once. (Hardest; requires coordination)

**Kafka exactly-once:**
- Producer idempotence: sequence numbers prevent duplicates from retries
- Transactional producer: atomically write to multiple partitions
- Consumer: commit offset and process atomically (using transactions)

**Database upserts for at-least-once consumers:**
```sql
-- Safe with duplicate delivery:
INSERT INTO processed_events (event_id, result)
VALUES (:event_id, :result)
ON CONFLICT (event_id) DO NOTHING;
```

---

## Interview Discussion Points

- "For this cross-service transaction, I'd avoid 2PC because it's blocking and creates tight coupling. Instead, I'd use the Saga pattern with the outbox for reliable event delivery."
- "The outbox pattern is key: it atomically ties the DB write and event publication together without a distributed transaction."
- "Every API that processes payments or creates resources needs idempotency keys. Clients must retry with the same key to avoid duplicate charges."
- "The Saga compensating transactions must be idempotent and eventually succeed — they can't fail permanently. If a compensation fails, we alert on-call and may need manual intervention."
- "For this order workflow that spans multiple days (payment → warehouse → shipping), I'd use Temporal. It handles retries, state persistence, and long waits natively."
