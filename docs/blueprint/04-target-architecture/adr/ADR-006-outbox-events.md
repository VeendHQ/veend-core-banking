# ADR-006: Transactional outbox with signed webhooks (Kafka optional later)

- Status: Proposed [R]

## Context

- Fineract uses an outbox (`m_external_event`) with Avro over Kafka or JMS ([V]).
- veend-tenant currently **polls** Fineract (`getLoan`, the aging report) to learn about state changes ([V] D2.3). It has no message broker, but it does have a webhook-receiving infrastructure (`Webhooks` collection, [V]).
- The framework has a `sendWebhook` Bull job ([V] `src/services/queue/workers.ts`).

## Decision

- Write the `outbox_event` row in the command transaction. A dispatcher worker uses `FOR UPDATE SKIP LOCKED` plus `LISTEN/NOTIFY` and fans each event out to webhook subscriptions through Bull.
- Payloads are JSON with a versioned `type` (e.g. `loan.repayment_posted.v1`), signed with HMAC-SHA256, and delivered at least once with exponential backoff and a DLQ.
- Kafka/Avro is a later, additive sink (P9) if external consumers need it.

## Consequences

- veend-tenant can replace polling jobs (`syncLoansFrom`, `checkOverdueTenureLoans`, closure detection inside the payroll jobs) with event handlers in M4 (D6).
- Consumers must deduplicate on event `id`.
