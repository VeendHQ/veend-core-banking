# Module: Loan Transactions (repayment, allocation, reversal, adjustments)

> Phase: P3 · Priority: P0 · Status: Spec draft
> Fineract reference: `LoanTransactionsApiResource:578-616`, `LoanAccountDomainServiceJpa.makeRepayment`, `LoanTransaction`, `LoanTransactionType` ([V] `:26-78`), `transactionprocessor/impl/*` (`FineractStyleLoanRepaymentScheduleTransactionProcessor` = `mifos-standard-strategy`), `m_loan_transaction_repayment_schedule_mapping`
> veend-tenant touchpoints: `makeRepayment` (all repayment channels; `paymentTypeId:1`, date = today), `undoRepayment` (match by amount) ([V] D2.2 T1–T2, D2.3 F3–F4)

## 1. Overview & business purpose

This module records every post-disbursement monetary event on a loan, allocates it to installment components using the product's strategy, keeps the summary and status current, and posts the matching journal entries. It is the module most exposed to the double-posting and value-date defects found in D2 (DF-01, DF-02, DF-05).

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-TXN-01 | **Repayment:** `amount > 0`, `valueDate` (default = business date; ≤ business date; ≥ disbursement date), `paymentType` (code), `externalRef` (required on the native API), `note`. Returns the transaction id, the allocation, the new status and outstanding |
| FR-TXN-02 | **Exactly-once:** `UNIQUE(payment_type_id, external_ref)` plus `Idempotency-Key` |
| FR-TXN-03 | **Allocation** by strategy (MVP: `mifos-standard-strategy` = Penalties → Fees → Interest → Principal per installment, oldest due first, then future installments in order, then overpayment). Other strategies are added as live products require [A] |
| FR-TXN-04 | **Backdated** transactions trigger replay of later transactions (see [processing §2](../04-target-architecture/processing.md#2-loan-transaction-processing-the-replay-model)) with delta postings |
| FR-TXN-05 | **Reversal** by transaction id, or by `(paymentType, externalRef)`. Marks the original reversed, replays later transactions, posts contra-entries, and may reopen a CLOSED loan. Policy guards: reversal window and maker-checker threshold |
| FR-TXN-06 | **Overpayment** → OVERPAID(700), with the excess held in the OVERPAYMENT liability. **Refund** of overpayment (by cash or transfer reference) |
| FR-TXN-07 | **Waive interest** (amount, date) and **waive charges** (see [charges](charges.md)) |
| FR-TXN-08 | **Goodwill credit / merchant refund**: not in MVP |
| FR-TXN-09 | Transaction list and detail with allocation |
| FR-TXN-10 | Compat T1: `POST /loans/{id}/transactions?command=repayment` returns `{officeId, clientId, loanId, resourceId, changes:{transactionDate, transactionAmount, …}}`. T2: `POST /loans/{id}/transactions/{txId}?command=undo` |

## 3. Features

| Feature | MVP? |
|---|---|
| Repayment (all channels via payment type) | Yes |
| Backdated repayment with replay | Yes |
| Reversal (by id or ref), reopen | Yes |
| Overpayment and refund | Yes |
| Interest waiver | Yes |
| Prepayment / foreclosure quote and payoff | Yes (via the payoff-quote and repayment of the quote) |
| Recovery repayment (after write-off) | Yes (see [writeoff-provisioning](writeoff-provisioning.md)) |
| Re-age, re-amortize | Yes (P7-R, [restructuring-rescheduling](restructuring-rescheduling.md)) |
| Charge-back, merchant refund, payout refund, goodwill | No (P9) |
| Bulk repayment import | Yes: `POST /v1/repayments/batch` (up to 500 items, each idempotent; per-item results) for payroll files |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-TXN-01 | `(payment_type, external_ref)` is unique per tenant **forever**. A duplicate returns the original transaction (compat: 200 with the original `resourceId`; native: 409 `DUPLICATE_EXTERNAL_REF` with a link) |
| BR-TXN-02 | `valueDate ≤ businessDate` and `≥ actualDisbursementDate`, and not in a closed GL period. Older than `max_backdate_days` needs approval |
| BR-TXN-03 | Repayments are only allowed on ACTIVE(300) or OVERPAID(700) loans. On WRITTEN_OFF(601) they are routed to a recovery repayment |
| BR-TXN-04 | **mifos-standard-strategy:** for installments in due-date order where `due_date ≤ valueDate` (in arrears or due), pay penalties → fees → interest → principal of that installment before the next. Then apply the remainder to future installments in order (same component order). Any remainder becomes overpayment. **Golden-test against Fineract.** |
| BR-TXN-05 | Rounding is not needed at allocation (amounts are already currency-scaled). The input amount is rounded to currency digits on intake; a sub-minor-unit amount is rejected |
| BR-TXN-06 | **Replay determinism:** order by `(value_date, booking sequence id)`. Re-allocation must equal what would have happened had the transactions arrived in value-date order |
| BR-TXN-07 | **Reversal policy:** by default allowed within `reversal_window_days` (30) without approval when the amount is ≤ threshold; otherwise maker-checker. Remita/IPPIS-sourced reversals are allowed only for OPS_SUPERVISOR (the current UI blocks them entirely; [V] `Loans.js:4557`) |
| BR-TXN-08 | Installment completion: `completed_on` is the value date of the transaction that zeroed it. It is reset on reversal |
| BR-TXN-09 | Inline COB catch-up runs before processing if `loan.last_cob_date < cobDate` |

## 5. Data model

`loan_transaction` (see [data-architecture](../04-target-architecture/data-architecture.md#3-core-tenant-schema-mvp)) plus:

```sql
CREATE TABLE loan_txn_allocation (txn_id BIGINT REFERENCES loan_transaction(id), installment_id BIGINT REFERENCES loan_installment(id),
  principal NUMERIC(19,6) DEFAULT 0, interest NUMERIC(19,6) DEFAULT 0, fee NUMERIC(19,6) DEFAULT 0, penalty NUMERIC(19,6) DEFAULT 0,
  PRIMARY KEY (txn_id, installment_id));
CREATE TABLE loan_txn_charge_paid (txn_id BIGINT REFERENCES loan_transaction(id), loan_charge_id BIGINT REFERENCES loan_charge(id),
  installment_number SMALLINT, amount NUMERIC(19,6) NOT NULL);
CREATE TABLE loan_txn_relation (from_txn_id BIGINT, to_txn_id BIGINT, relation TEXT CHECK (relation IN ('REVERSAL','REPLAYED','TOPUP_CLOSE','REFUND_OF')),
  PRIMARY KEY (from_txn_id, to_txn_id, relation));
CREATE INDEX ix_ltx_loan_date ON loan_transaction(loan_id, value_date, id) WHERE NOT reversed;
```

## 6. API endpoints

**Native:**
- `POST /v1/loans/{id}/repayments`, `POST /v1/repayments/batch`
- `GET /v1/loans/{id}/transactions`, `GET /v1/loans/{id}/transactions/{txId}`, `GET /v1/transactions?paymentType=&externalRef=`
- `POST /v1/loans/{id}/transactions/{txId}/reversal`, `POST /v1/loans/{id}/transactions/reversal-by-ref`
- `POST /v1/loans/{id}/refunds` (overpayment)
- `POST /v1/loans/{id}/waivers/interest`

**Compat:** T1 and T2 as in D2.2.
- T1 accepts `paymentTypeId` (1 → MANUAL by default; mapping table per tenant), `transactionDate`, `note`, and optional `externalId` (a Fineract field), which maps to `externalRef`.
- **Derived idempotency** when `externalId` is absent: `sha256(loanId|amount|transactionDate|note|paymentTypeId)` deduplicates within 24 h. This protects against DF-02 until veend-tenant sends references (M2). [R] A legitimate identical second payment on the same day with the same note is rare. veend-tenant's payroll references already differ by deduction date.

## 7. Service-layer responsibilities

- Commands: `MakeRepayment`, `ReverseTransaction`, `RefundOverpayment`, `WaiveInterest`, `BatchRepayments` (fan-out with per-item transactions).
- Domain: `AllocationStrategy` (interface: `allocate(loanState, txn) → allocations`), `MifosStandardStrategy`, `ReplayEngine`, `PostingDeltaCalculator`.

## 8. Dependencies

loan-lifecycle (aggregate), schedule engine (installment dues), charges, accounting, organisation (payment types), COB (inline catch-up).

## 9. Integration

Events: `loan.repayment_posted` (with allocation and `externalRef`), `loan.transaction_reversed`, `loan.closed`, `loan.overpaid`, `loan.reopened`. veend-tenant uses these to update its mirror and stop mandates (M4), replacing `payoff`'s local closure logic.

## 10. Authorization

`REPAYMENT_LOAN`, `ADJUST_LOAN` (reversal), `REFUND_LOAN`, `WAIVEINTERESTPORTION_LOAN`. Maker-checker on `ADJUST_LOAN` beyond the policy window or threshold.

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Repayment | Clearing (payment type) or FUND_SOURCE | LOAN_PORTFOLIO (principal), INTEREST_RECEIVABLE (interest), FEES_RECEIVABLE (fee), PENALTIES_RECEIVABLE (penalty), OVERPAYMENT (excess) |
| Repayment (CASH rule) | Clearing | LOAN_PORTFOLIO, INTEREST_ON_LOANS, INCOME_FROM_FEES, INCOME_FROM_PENALTIES, OVERPAYMENT |
| Reversal | Mirror of the original | |
| Replay delta | Only changed component amounts (e.g. principal → interest reclass: Dr LOAN_PORTFOLIO / Cr INTEREST_RECEIVABLE) | |
| Refund overpayment | OVERPAYMENT | Clearing |
| Interest waiver (accrual) | INTEREST_ON_LOANS | INTEREST_RECEIVABLE |

All lines carry `loan_id` and `client_id`. The entry date is the value date.

## 12. Error handling & edge cases

| Case | Behaviour |
|---|---|
| Same provider ref twice (webhook redelivery) | BR-TXN-01: the original is returned. Logged as a duplicate |
| Concurrent repayments on the same loan | Serialised by `FOR UPDATE`. Both succeed in order |
| Repayment on an APPROVED (not disbursed) loan | 409 `INVALID_STATE` |
| Backdated repayment before a penalty's trigger date | Replay removes the penalty, with an ACCRUAL_ADJUSTMENT posting (see charges §12) |
| Reverse a transaction on a CLOSED loan | Reopens to ACTIVE; emits `loan.reopened` |
| Reverse when a later transaction has been reversed already | Replay considers only non-reversed transactions |
| Value date in a closed period | 423 `PERIOD_CLOSED` |
| Repayment amount with 3 decimals for NGN | 400 |
| Remita deduction posted twice by veend-tenant with the same `${loanId}_${date}` | The facade replays the first (derived key) and returns the same `resourceId` |

## 13. Testing

- Golden fixtures: allocation sequences (on time, early, partial, late, overpay, multiple installments, backdated, reversal) for each live strategy, kobo-exact vs Fineract.
- Property tests: Σ allocations = amount − overpayment; outstanding never negative; replay(order permutation by value date) is idempotent.
- Concurrency: 50 parallel repayments with distinct refs give 50 transactions and correct totals; 50 parallel with the **same** ref give 1 transaction.
- Compat: recorded `makeRepayment` and `undoRepayment` payloads.

## 14. Status & gaps

- Not implemented.
- [A] The strategy codes of live products.
- [A] Whether any loans in production use interest recalculation (which changes allocation semantics).
- [A] Finance policy on reversal windows and thresholds.
