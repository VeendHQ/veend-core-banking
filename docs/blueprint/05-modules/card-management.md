# Module: Card Management

> Phase: **P7-K** · Priority: P2 · Status: Spec draft
> Scope origin: F-57 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.7 "connect card profiles, card/cash disputes and card-related reports to the relevant customer accounts and payment transactions"
> Fineract reference: none. Related existing code: Giro `Cardrequests`, `Cardprograms`, `Cardfiles`, `services/vervecard` ([V] `bill-payment-portal/controllers`, `services`)

## 1. Overview & business purpose

This module manages debit/prepaid cards linked to CASA accounts:
- card programmes and products;
- card requests, issuance files and activation;
- PIN and limits;
- block/unblock and replacement;
- authorisation and settlement postings received from the card processor/switch;
- **card and cash (ATM) disputes**;
- card reports.

Card data is tokenised. Fineract-Mod never stores a full PAN or PIN (PCI-DSS scope minimisation).

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-CRD-01 | Card programmes/products: scheme (Verve, Mastercard, Visa), BIN range, fees (issuance, maintenance), limits (ATM, POS, web, per day), linked account types |
| FR-CRD-02 | Card profile: customer, linked CASA, token/masked PAN, expiry, status (requested, issued, active, blocked, hot-listed, expired), channels enabled |
| FR-CRD-03 | Requests and issuance: request → approval → card file to the bureau/processor → delivery → activation. Bulk requests (e.g. staff or payroll cards) |
| FR-CRD-04 | Authorisation interface with the processor: balance enquiry, hold on authorisation, release/settle on clearing (holds via [savings-current-accounts](savings-current-accounts.md) FR-SAV-06) |
| FR-CRD-05 | Settlement: daily scheme/switch settlement file → postings → reconciliation (registered control) |
| FR-CRD-06 | **Disputes**: customer card or ATM cash disputes (dispense errors, double debits, fraud) → logging, chargeback/arbitration workflow with the switch, provisional credit (optional), resolution and posting |
| FR-CRD-07 | Card controls: block/unblock, limit changes, channel toggles, replacement/renewal |
| FR-CRD-08 | Reports: cards issued, active, by status, card fees, transactions, dispute ageing |

## 3. Features & sub-features

Programmes · profiles · requests and issuance · authorisation holds · settlement · disputes · controls · reports.

## 4. Business rules & validations

BR-CRD-01: PAN is tokenised by the processor, and only the token plus last 4 digits are stored. BR-CRD-02: an authorisation hold expires at T+N days if no clearing arrives [A scheme rules]. BR-CRD-03: dispute SLAs follow CBN consumer-protection timelines [A]. BR-CRD-04: a provisional credit is reversed if the dispute is lost.

## 5. Data model

```sql
CREATE TABLE card_product (id BIGSERIAL PRIMARY KEY, scheme TEXT, bin TEXT, name TEXT, fees JSONB, limits JSONB, active BOOLEAN);
CREATE TABLE card (id BIGSERIAL PRIMARY KEY, product_id BIGINT, client_id BIGINT, account_id BIGINT, token TEXT UNIQUE, last4 CHAR(4),
  expiry CHAR(4), status TEXT, limits JSONB, issued_on DATE);
CREATE TABLE card_request (id BIGSERIAL PRIMARY KEY, client_id BIGINT, product_id BIGINT, account_id BIGINT, status TEXT, batch_ref TEXT);
CREATE TABLE card_auth (id BIGSERIAL PRIMARY KEY, card_id BIGINT, stan TEXT, rrn TEXT, amount NUMERIC(19,6), hold_id BIGINT, status TEXT, UNIQUE (rrn, stan));
CREATE TABLE card_dispute (id BIGSERIAL PRIMARY KEY, card_id BIGINT, auth_id BIGINT, type TEXT, amount NUMERIC(19,6), status TEXT,
  provisional_credit_txn BIGINT, opened_on DATE, resolved_on DATE, resolution TEXT);
```

## 6. API endpoints

`GET/POST /v1/card-products`, `POST /v1/card-requests`, `POST /v1/card-requests/batch`, `GET /v1/cards/{id}`, `POST /v1/cards/{id}/block|unblock|replace`, `PATCH /v1/cards/{id}/limits`, `POST /internal/card-processor/authorisations` (mTLS), `POST /v1/card-settlements` (file), `GET/POST /v1/card-disputes`, `POST /v1/card-disputes/{id}/resolve`.

## 7. Service-layer responsibilities

`CardLifecycleService`, `ProcessorAdapter` (authorisation, issuance files), `CardSettlementPoster`, `DisputeWorkflow` (shared with [payments disputes](payments-transfers-bills.md) FR-PAY-12).

## 8. Dependencies

clients, savings-current-accounts (holds, debits), charges, accounting, reconciliation-exceptions, payments-transfers-bills.

## 9. Integration requirements

Card processor/switch (e.g. the Interswitch/Verve ecosystem) [A]; card bureau. Giro's existing card programme integration may be reused or migrated [A].

## 10. Authorization & permissions

`MANAGE_CARDPRODUCTS`, `REQUEST_CARD`, `APPROVE_CARDREQUEST` (`_CHECKER`), `BLOCK_CARD`, `RESOLVE_CARDDISPUTE` (+ `_CHECKER` for credits).

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Auth hold | (no posting; hold only) | |
| Clearing (POS/ATM) | SAVINGS_CONTROL | CARD_SETTLEMENT (switch) |
| Card fee | SAVINGS_CONTROL | INCOME_FROM_FEES |
| Provisional credit | DISPUTE_SUSPENSE | SAVINGS_CONTROL |
| Dispute won (switch refunds) | CARD_SETTLEMENT | DISPUTE_SUSPENSE |

## 12. Error handling & edge cases

Clearing without an auth (offline), partial reversals, duplicate settlement lines (unique RRN/STAN), and an expired hold followed by late clearing.

## 13. Testing requirements

Processor simulator; settlement reconciliation; PCI scope review (no PAN in logs/DB).

## 14. Implementation status & outstanding gaps

Not implemented. [A] Processor and scheme choice; whether Pettysave or Giro currently issue cards and on which platform.
