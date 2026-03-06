# System Design: Google Docs (Collaborative Editing)

## Problem

Design a collaborative document editor where multiple users edit the same document in real time with minimal conflict and low latency. Changes should be merged and visible to all participants.

## Requirements

**Functional:** Create/edit document; real-time sync of edits; multiple cursors; presence (who’s viewing); optional comments and history.

**Non-functional:** Low latency for sync (sub-second); eventual consistency with minimal visible conflict; handle offline and reconnect.

## High-Level Design

```
  Clients (browser) ↔ WebSocket ↔ Collaboration service ↔ Document store + OT/CRDT layer
       ↑                              ↑
  Operational Transform (OT) or CRDT for merge
```

- **Edit flow:** User types → client generates operation (e.g. “insert ‘x’ at position 5”); send to server; server transforms against concurrent ops; broadcast to other clients; clients apply and update view.
- **Merge:** OT (Operational Transform) or CRDT (Conflict-free Replicated Data Type). OT: server is authority; transforms operations. CRDT: deterministic merge on clients; no central transform.
- **Persistence:** Document state (or full history of ops) in DB; load on join; sync ongoing via WebSocket.

## Key Components

- **Collaboration service:** Accept ops from clients; transform (OT) or broadcast (CRDT); persist; broadcast to other clients in same doc. Stateful (session per doc or per doc shard).
- **Document store:** Current snapshot (for load) and optionally op log (for history and replay). Version or vector clock for ordering.
- **Client:** Apply ops locally (optimistic); send op to server; receive remote ops and merge (OT transform or CRDT merge); resolve cursor positions.
- **Presence:** Who is in doc; cursor/selection per user; broadcast with ops.

## Data Model

- **documents:** document_id, content (snapshot or structured), version. Optional: op_log (op_id, op, version).
- **ops:** For OT: operation type (insert/delete), position, content, user_id, timestamp or version.
- **CRDT:** Document as CRDT structure (e.g. tree or sequence with unique IDs); merge by rules.

## Scale

- Per-doc: tens of concurrent editors; ops per second per doc moderate. Service: many docs; shard by document_id. WebSocket connections scale with connection manager (e.g. per-doc room).

## Trade-offs

- **OT vs CRDT:** OT: server does transform; good for rich text; central. CRDT: no server transform; offline-friendly; sometimes more metadata (IDs per character).
- **Snapshot vs op log:** Snapshot: fast load; op log: history, replay, smaller incremental updates.
- **Consistency:** Strong per-doc with OT/CRDT; eventual consistency across clients; conflict resolution by design.

## Failure & Operations

- **Reconnect:** Client sends last known version; server sends missed ops or snapshot; client reconciles.
- **Conflict:** OT/CRDT designed to avoid user-visible conflict; rare cases: last-write-wins or manual resolve.
- **Monitoring:** Sync latency, op throughput, connection count, merge errors.
