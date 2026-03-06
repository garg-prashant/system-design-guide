# System Design: Chat System

## Problem

Design a real-time chat system: 1:1 and group messaging, delivery and read receipts, presence (online/offline), and message history. Scale to millions of users.

## Requirements

**Functional:** Send/receive messages (text, optional media); 1:1 and group chats; delivery/read receipts; presence; history (paginated); optional typing indicator.

**Non-functional:** Low latency (deliver in hundreds of ms); reliable (no lost messages); support offline (queue and sync on connect); scalable.

## High-Level Design

```
  Clients ↔ WebSocket / long poll ↔ Chat service ↔ Message store + Queue
       ↑                                ↑
  Presence (heartbeat, online list)   Message persistence, fanout to group
```

- **Send:** Client sends message to API; validate (member of conversation); persist message; fanout to group members (push to online, queue for offline); ack to sender.
- **Receive:** Online: push over WebSocket. Offline: on connect, fetch messages since last_sequence; push notifications for new messages (optional).
- **Presence:** Client heartbeat; update last_seen; list of online users per conversation or global; broadcast on change.
- **History:** Paginate by conversation_id, cursor (message_id or timestamp); return N messages before cursor.

## Key Components

- **Chat API / WebSocket server:** Accept connections; route messages; validate membership; persist; fanout. Stateful (connection per user); scale with connection managers (e.g. per room or shard).
- **Message store:** messages (message_id, conversation_id, sender_id, content, created_at, sequence). Shard by conversation_id. Index for history (conversation_id, created_at).
- **Fanout:** For each member: if online, push via WebSocket; else enqueue for later (notification or fetch on connect).
- **Presence:** Redis or in-memory: user_id → online, last_seen; TTL on heartbeat. Broadcast to relevant clients when someone comes online/offline.
- **Media:** Upload to object store; message contains URL; optional CDN for download.

## Data Model

- **messages:** message_id, conversation_id, sender_id, body, created_at, sequence.
- **conversations:** conversation_id, type (1:1/group), members[].
- **presence:** user_id → { online, last_seen }; TTL.
- **delivery:** message_id, user_id, delivered_at, read_at (optional).

## Scale

- Millions of concurrent connections; millions of messages/s. Shard messages by conversation; WebSocket servers by user_id or connection ID. Message queue for offline delivery; presence in Redis cluster.

## Trade-offs

- **WebSocket vs long poll:** WebSocket: efficient, low latency. Long poll: simpler, works everywhere.
- **Ordering:** Per-conversation sequence; single writer (conversation) or logical timestamp for ordering across shards.
- **History:** Store forever vs retention; paginate with cursor; optional search index.

## Failure & Operations

- **Durability:** Persist before ack to sender; then fanout. Retry for offline delivery; idempotent receive.
- **Reconnect:** Client reconnects; fetch since last_sequence; reconcile; optional idempotency key for sends.
- **Monitoring:** Delivery latency, message loss, connection count, queue depth, presence accuracy.
