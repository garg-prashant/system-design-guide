# System Design: WhatsApp

## Problem

Design a messaging system: send/receive text and media messages, 1:1 and group chats, delivery status, presence (online/offline), and optional end-to-end encryption.

## Requirements

**Functional:** Send/receive messages (text, media); 1:1 and group chats; delivery/read receipts; presence; optional E2E encryption.

**Non-functional:** Low latency (messages delivered in hundreds of ms); reliability (no lost messages); scale to billions of users; offline support (queue and sync).

## High-Level Design

```
  Clients (mobile) ↔ WebSocket / long poll ↔ Message service ↔ Queue + DB
       ↑                    ↑
  Presence service    Media service (upload/download via URL)
```

- **Send message:** Client → API → validate → store message in DB → enqueue for delivery to recipients (or push notification if offline).
- **Receive:** Long-lived connection (WebSocket or MQTT); server pushes new messages to online clients. Offline: client polls or connects and fetches since last_sequence.
- **Presence:** Heartbeat from client; store online status; broadcast to relevant users (e.g. contacts); ephemeral, low latency.
- **Media:** Upload to object store; message contains URL; download via CDN.

## Key Components

- **Message service:** Store messages (message_id, conversation_id, sender, payload, timestamp); sequence or cursor for ordering. Shard by conversation_id or user_id.
- **Delivery:** Online: push over WebSocket. Offline: store in DB; push notification; client fetches on connect (sync since seq).
- **Presence:** In-memory store (Redis) or dedicated service; TTL on heartbeat; broadcast changes to subscribers.
- **E2E encryption (optional):** Keys per device; server stores only ciphertext; key exchange (Signal protocol); server cannot decrypt.
- **Group:** Group metadata (members, name); message has conversation_id = group_id; fanout to members’ delivery queues.

## Data Model

- **messages:** message_id, conversation_id, sender_id, body (or media_url), created_at, sequence.
- **conversations:** conversation_id, type (1:1/group), members.
- **presence:** user_id → { online, last_seen }; TTL.
- **delivery state:** message_id, user_id, delivered_at, read_at (optional).

## Scale

- Billions of users; millions of messages/s. Shard messages by conversation or user; message queue per user or per conversation for delivery. Presence: ephemeral, scale with Redis cluster. Media: object store + CDN.

## Trade-offs

- **WebSocket vs long poll:** WebSocket: efficient, low latency. Long poll: simpler, works everywhere; more overhead.
- **Message ordering:** Per-conversation sequence; strong ordering in single server; across shards use logical timestamp or accept best-effort.
- **E2E:** Privacy vs server-side features (search, compliance); key management and multi-device are complex.

## Failure & Operations

- **Durability:** Persist message before ack to sender; then deliver; retry and idempotency for receivers.
- **Presence:** Accept eventual consistency; TTL and reconnect to clear stale.
- **Monitoring:** Delivery latency, message loss, connection count, queue depth.
