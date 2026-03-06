# System Design: Payment System

## Problem

Design a payment processing system: accept payment requests (e.g. charge card), ensure exactly-once and idempotency, integrate with payment providers, handle refunds and reconciliation, and meet compliance (PCI, audit).

## Requirements

**Functional:** Charge (amount, currency, idempotency key); refund; idempotent (same key = same result); support multiple payment methods (card, wallet); reconciliation with provider and internal ledger.

**Non-functional:** Strong consistency for balance and double-spend prevention; durable (no lost payments); audit trail; PCI compliance (no raw card storage); high availability.

## High-Level Design

```
  Client → API (idempotency key) → Payment service → Idempotency check → Ledger + Provider call
                                        ↓
  Ledger: append-only (credit/debit); balance = sum; Provider: Stripe, etc.
```

- **Charge:** Request (amount, idempotency_key, payment_method). Check idempotency: if key seen, return stored response. Else: reserve or charge via provider; write to ledger (debit user, credit merchant); store response under idempotency key; return.
- **Refund:** Same idempotency pattern; credit user, debit merchant; call provider refund.
- **Ledger:** Append-only; each row = transaction_id, account_id, amount, type (charge/refund), idempotency_key, created_at. Balance = SUM(amount) per account. No updates; only inserts.
- **Provider:** Stripe, Adyen, etc.; tokenize card (never store raw); charge/refund via API; webhook for async status; reconcile daily.

## Key Components

- **Payment API:** Validate request; check idempotency (DB or cache); call payment service; return response. Idempotency key stored with response and TTL (e.g. 24h).
- **Payment service:** Call provider (charge/refund); on success, write ledger; store idempotency result; handle provider errors (retry, manual review).
- **Ledger:** Append-only store (DB table or event log); balance computed or materialized; strong consistency (single writer or transactional).
- **Reconciliation:** Daily: fetch provider settlement; match with ledger; flag discrepancies; alert.
- **Fraud (optional):** Rules or ML before charge; velocity checks, blocklist.

## Data Model

- **idempotency:** key (idempotency_key), response, status, created_at. Unique constraint.
- **ledger:** transaction_id, account_id, amount, currency, type, idempotency_key, provider_ref, created_at.
- **accounts:** account_id, balance (materialized or computed), updated_at.

## Scale

- Throughput: thousands of TPS; ledger append-only scales; idempotency lookup by key (indexed). Provider calls are bottleneck; queue and retry with backoff.

## Trade-offs

- **Strong consistency:** Ledger and balance in same DB with transaction; avoid double-spend. Eventual consistency not acceptable for money.
- **Idempotency:** Client must send key; server stores result; same key returns same result. Critical for retries and duplicate requests.
- **Provider:** Outsource card handling (PCI); own ledger for audit and reconciliation.

## Failure & Operations

- **Provider down:** Queue charge requests; retry with backoff; return “pending” or timeout; don’t double-charge (idempotency).
- **Reconciliation:** Automated daily; alert on mismatch; manual resolution process.
- **Monitoring:** Success/failure rate, latency, ledger balance vs provider, reconciliation status.
