# System Design: Distributed Job Scheduler

## Problem

Design a distributed job scheduler: schedule recurring or one-off jobs (cron-like), assign jobs to workers, ensure at-least-once or exactly-once execution, and handle failures and scale. Think cron at scale across many machines.

## Requirements

**Functional:** Schedule job (cron expression or one-off at time); assign to worker; execute job (run script or call API); retry on failure; support dependencies (job B after job A); optional priority and resource limits.

**Non-functional:** Reliable (no missed schedules under normal conditions); at-least-once execution (retry); scale to thousands of jobs and workers; avoid duplicate execution (optional exactly-once).

## High-Level Design

```
  Scheduler (leader) → Reads schedule → Triggers at time → Enqueue job to queue
       ↑                                                                  ↓
  Leader election (etcd, etc.)                              Workers poll or consume → Execute job → Ack/fail
```

- **Scheduling:** Single leader (e.g. elected via etcd/ZooKeeper) reads job definitions and cron; at each tick (e.g. every second), find jobs due; enqueue to job queue (e.g. Kafka, Redis, DB queue). Or: distributed cron (each scheduler instance owns a shard of jobs).
- **Queue:** Partition by job type or priority; workers consume; one consumer per partition for ordering; at-least-once: ack after execution; retry on failure (re-queue or dead-letter).
- **Execution:** Worker pulls job; executes (subprocess, HTTP call, or container); reports success/failure; idempotency key to dedupe if re-queued.
- **Leader election:** Only leader triggers schedules; avoid duplicate triggers; failover when leader dies.

## Key Components

- **Job store:** job_id, schedule (cron or run_at), payload, retry_policy, status. Index: next_run_time for “due” query.
- **Scheduler process:** Leader reads due jobs; enqueues with job_id, payload, idempotency_key; updates next_run_time (or separate trigger log). Scale: one active leader; standby for failover.
- **Job queue:** Kafka, Redis Streams, or DB-backed queue; partition for parallelism; consumer groups for workers.
- **Workers:** Stateless; consume job; execute; ack. On failure: nack and retry (with backoff); after N retries, dead-letter. Idempotent execution when possible.
- **Leader election:** etcd, ZooKeeper, or DB-based lease; single leader at a time.

## Data Model

- **jobs:** job_id, schedule, payload, next_run_time, status, retry_count, created_at.
- **executions:** execution_id, job_id, started_at, finished_at, status, idempotency_key. For audit and dedup.
- **queue:** job_id, payload, idempotency_key (in message).

## Scale

- Thousands of jobs; trigger rate bounded by cron granularity (e.g. 1-min cron = at most 1 trigger per job per minute). Queue partitions and workers scale; scheduler single leader is usually enough (or shard jobs across leaders with consistent assignment).

## Trade-offs

- **At-least-once vs exactly-once:** At-least-once: simple; idempotent job logic. Exactly-once: dedup by idempotency_key; or single consumer per partition + transactional outbox.
- **Leader single point:** One leader simplifies scheduling; use leader election for HA. Alternative: distributed cron (each node owns job shard) for scale.
- **Queue choice:** Kafka: durable, replay. Redis: low latency, less durability. DB: simple, transactional; can be bottleneck.

## Failure & Operations

- **Worker failure:** Job re-queued (nack or timeout); retry with backoff; dead-letter after N failures; alert.
- **Leader failure:** New leader elected; resume reading due jobs; may trigger same job again (idempotency).
- **Monitoring:** Schedule lag, job success/failure rate, queue depth, worker utilization, leader identity.
