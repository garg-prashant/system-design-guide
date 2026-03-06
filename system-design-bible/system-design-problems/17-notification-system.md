# System Design: Notification System

## Problem

Design a system to send notifications to users via multiple channels: push (mobile), email, SMS, and in-app. Notifications are triggered by events (e.g. order shipped, mention); support preferences and batching.

## Requirements

**Functional:** Trigger notification by event (user_id, channel, template, payload); support push, email, SMS, in-app; user preferences (opt-in per channel); optional batching/digest.

**Non-functional:** Deliver in seconds to minutes; high throughput (millions/day); respect rate limits (carriers, email); reliability (retry, dead-letter).

## High-Level Design

```
  Events (order, mention, etc.) → Notification service → Router (by channel + preferences)
                                                                  ↓
  Push: FCM/APNs  |  Email: SES/SendGrid  |  SMS: Twilio  |  In-app: WebSocket / DB
```

- **Trigger:** Other services publish event (e.g. “order_shipped”, user_id, order_id); notification service consumes.
- **Routing:** Resolve template (e.g. “Your order {order_id} shipped”); check user preferences (channel, frequency); optionally batch (digest); enqueue per channel.
- **Delivery:** Workers per channel: push (call FCM/APNs), email (SMTP or API), SMS (provider API), in-app (write to DB + push via WebSocket if online).
- **Retry:** Failed delivery → retry with backoff; dead-letter after N failures; alert.

## Key Components

- **Notification service:** Consume events; resolve template; check preferences; enqueue to channel queues. Idempotent (event_id) to avoid duplicate sends.
- **Preferences store:** user_id → channels enabled, frequency (instant vs digest). Consult before send.
- **Template store:** template_id → body, subject (email); variables. Render with event payload.
- **Channel workers:** Push worker (FCM/APNs); email worker (SES); SMS worker (Twilio); in-app worker (write to notifications table, optional push).
- **In-app delivery:** Table (user_id, notification_id, read); API to list/poll; optional WebSocket push when user online.

## Data Model

- **events:** event_id, type, user_id, payload, created_at.
- **preferences:** user_id, channel, enabled, frequency.
- **templates:** template_id, channel, body, subject.
- **notifications (in-app):** notification_id, user_id, content, read, created_at.
- **delivery_log:** event_id, channel, status, sent_at (for idempotency and audit).

## Scale

- Millions of events per day; fan-out to multiple channels. Queue per channel; workers scale with queue depth. Rate limits: throttle per provider; batch where possible (email digest).

## Trade-offs

- **Sync vs async:** Async (queue) for scale and reliability; sync only for simple “send one” API.
- **Batching:** Digest reduces email/SMS volume and cost; increases latency. Per-event for critical; digest for non-urgent.
- **Channels:** Third-party for push/email/SMS (Twilio, FCM, SES); in-app is own storage and optional real-time push.

## Failure & Operations

- **Retry and DLQ:** Exponential backoff; dead-letter for manual review; alert on DLQ depth.
- **Rate limits:** Respect provider limits; queue and throttle; backpressure to event source if needed.
- **Monitoring:** Delivery rate, latency per channel, failure rate, provider health.
