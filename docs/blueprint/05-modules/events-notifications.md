# Module: Events and Notifications (outbox, webhooks)

> Phase: P5 (the outbox itself is P0) · Priority: P1 · Status: Spec draft
> Fineract reference: `ExternalEventService`, `m_external_event`, `m_external_event_configuration`, `SendAsynchronousEventsTasklet`, Avro `MessageV1` ([V] D1.1 §8); webhooks `m_hook`
> veend-tenant touchpoints: none today. veend-tenant polls the CBA (`syncLoansFrom`, the closure check in the payroll jobs, the Aging report for arrears). It has webhook ingestion infrastructure (`Webhooks` model, `controllers/Webhooks.js`) ([V])

## 1. Overview & business purpose

This module reliably tells downstream systems (veend-tenant first) about CBA state changes, so they stop polling and mirroring balances. **Borrower notifications (SMS/email) stay in veend-tenant**: the CBA emits facts, and veend-tenant decides what to tell the borrower.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-EVT-01 | Outbox insert in the command transaction (ADR-006) |
| FR-EVT-02 | Event catalogue with JSON Schemas, versioned (`type: "loan.repayment_posted", version: 1`) |
| FR-EVT-03 | Webhook subscriptions per tenant: URL, secret, event filter, active flag, and per-tenant on/off by event type (Fineract `m_external_event_configuration` equivalent) |
| FR-EVT-04 | Delivery: HMAC-SHA256 signature, at least once, exponential backoff (1m → 12h, 15 attempts), DLQ, manual redeliver, test ping |
| FR-EVT-05 | Ordering per aggregate (loan id) within a subscription |
| FR-EVT-06 | A replay API to re-send events for a time range or aggregate (for consumer recovery) |
| FR-EVT-07 | Delivery log retained 30 days |

## 3. Event catalogue (MVP)

| Type | Key payload |
|---|---|
| `client.created` / `client.updated` / `client.closed` | `clientId, externalId, accountNo, changes` |
| `loan.submitted` / `loan.approved` / `loan.approval_undone` / `loan.rejected` / `loan.withdrawn` | `loanId, externalId, clientId, productId, principal, status` |
| `loan.disbursed` / `loan.disbursement_undone` | `loanId, amount, netDisbursed, feesAtDisbursement, valueDate, externalRef, schedule summary` |
| `loan.repayment_posted` | `loanId, transactionId, externalRef, paymentType, amount, valueDate, allocation{…}, outstanding{…}, status` |
| `loan.transaction_reversed` | `loanId, transactionId, reversalOf, externalRef` |
| `loan.closed` / `loan.overpaid` / `loan.reopened` | `loanId, status, closedOn, overpaidAmount` |
| `loan.installment_due` / `loan.overdue` / `loan.delinquency_changed` / `loan.npa_changed` | `loanId, dpd, bucket, overdue{…}` |
| `loan.written_off` / `loan.writeoff_undone` / `loan.recovery_posted` | `loanId, amount, date` |
| `loan.charge_applied` / `loan.charge_waived` | `loanId, chargeId, amount` |
| `deposit.opened` / `deposit.funded` / `deposit.activated` / `deposit.expired` / `deposit.payout_due` / `deposit.payout_settled` / `deposit.matured` / `deposit.closed` / `deposit.deployed` | `depositAccountId, externalId, amount, dueDate, status` (subscriber: Giro) |
| `journal.posted` | `entryId, date, lines` (finance subscribers only) |
| `cob.completed` | `cobDate, stats` |

Envelope:

```json
{ "id": "evt_01JC…", "type": "loan.repayment_posted", "version": 1, "tenant": "lenderA",
  "occurredAt": "2026-11-02T10:15:30Z", "businessDate": "2026-11-02", "commandId": 88123,
  "aggregate": {"type": "loan", "id": 48211, "externalId": "65f1c…"}, "data": { … } }
```

Headers: `X-FM-Event-Id`, `X-FM-Event-Type`, `X-FM-Signature: t=1730542530,v1=<hex(hmac_sha256(secret, t + "." + body))>`.

## 4. Business rules

- **BR-EVT-01.** Events are emitted only for committed state, which the outbox guarantees.
- **BR-EVT-02.** Consumers must deduplicate by `id`. The CBA never reuses ids.
- **BR-EVT-03.** A subscription that has been continuously failing for 24 h is auto-paused, and alerts fire.

## 5. Data model

```sql
CREATE TABLE outbox_event (id BIGSERIAL PRIMARY KEY, event_uid TEXT UNIQUE NOT NULL, type TEXT NOT NULL, version SMALLINT NOT NULL,
  aggregate_type TEXT, aggregate_id BIGINT, payload JSONB NOT NULL, command_id BIGINT, created_at TIMESTAMPTZ DEFAULT now(), dispatched_at TIMESTAMPTZ);
CREATE INDEX ix_outbox_pending ON outbox_event (id) WHERE dispatched_at IS NULL;
CREATE TABLE webhook_subscription (id BIGSERIAL PRIMARY KEY, url TEXT NOT NULL, secret_enc BYTEA NOT NULL, event_types TEXT[] NOT NULL,
  active BOOLEAN DEFAULT true, paused_reason TEXT);
CREATE TABLE webhook_delivery (id BIGSERIAL PRIMARY KEY, subscription_id BIGINT, outbox_id BIGINT, attempt INT, status TEXT, http_status INT,
  response_snippet TEXT, next_attempt_at TIMESTAMPTZ, created_at TIMESTAMPTZ DEFAULT now(), UNIQUE (subscription_id, outbox_id, attempt));
```

## 6. API endpoints

- `GET/POST /v1/webhook-subscriptions`, `PATCH/DELETE /v1/webhook-subscriptions/{id}`, `POST /v1/webhook-subscriptions/{id}/test`
- `GET /v1/webhook-deliveries?status&subscription`, `POST /v1/webhook-deliveries/{id}/redeliver`
- `POST /v1/events/replay` (`{from, to, types?, aggregateId?, subscriptionId}`)
- `GET /v1/events/schemas`

## 7. Service-layer responsibilities

`Outbox.add()` (called by handlers), `OutboxDispatcher` (worker), `WebhookDeliverer` (reuses the framework's Bull `sendWebhook` pattern, hardened), `Signer`, `SubscriptionService`.

## 8. Dependencies

platform-core. All domain modules produce events.

## 9. Integration

veend-tenant needs a new endpoint `POST /cba/events` that verifies the signature, deduplicates via an `event_uid` unique index, and dispatches to handlers (M4). A Kafka sink is P7.

## 10. Authorization

`CREATE/UPDATE/READ_WEBHOOK`, `REPLAY_EVENTS`.

## 11. Transactional & accounting implications

None. Events describe postings already committed.

## 12. Edge cases

- Consumer down for days: deliveries go to the DLQ after 15 attempts, and ops replays by range.
- Event schema evolution: add fields only; a breaking change means a `version` bump with dual emission for 3 months.

## 13. Testing

Outbox/commit atomicity (kill between commit and dispatch: the event is still delivered). Signature verification test vectors. Ordering per loan under concurrency.

## 14. Status & gaps

Not implemented. The framework has a basic `sendWebhook` job ([V] `src/services/queue/workers.ts`) with a 5-minute re-queue that can deliver twice ([V] D1.3 G5). It will be replaced with the design above.
