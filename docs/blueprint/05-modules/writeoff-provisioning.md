# Module: Write-off, Recovery, Charge-off and Provisioning

> Phase: P4 · Priority: P0 (fixes DF-04) · Status: Spec draft
> Fineract reference: `LoanTransactionsApiResource` commands `writeoff`, `undowriteoff`, `recoverypayment`, `chargeOff`, `undoChargeOff` ([V] `:592-610`); `ChargeOffLoanCommandHandler`; `AccrualBasedAccountingProcessorForLoan.createJournalEntriesForWriteOffs` (`:1379`), `createJournalEntriesForRecoveryRepayments` (`:1983`); `ProvisioningCriteria`, `ProvisioningCategory` (seed: STANDARD, SUB-STANDARD, DOUBTFUL, LOSS; [V] `0002_initial_data.xml:10141-10156`), `ProvisioningEntry`
> veend-tenant touchpoints: `jobs.writeOffLoan` (local flag only), `jobs.insuranceClaimed` (local only), `markLoanDefaulted` ([V] D2.3 F7)

## 1. Overview & business purpose

This module recognises credit losses correctly in the books. It covers:
- **write-off**: derecognising the outstanding balance as a loss
- **recovery**: income from collections after write-off, including insurance claims
- **charge-off**: an accounting classification that keeps the loan collectable while moving it off performing income
- **provisioning**: a periodic loan-loss allowance by aging category

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-WO-01 | **Write-off** a loan (status 300 → 601) as of a date, with a reason code and note. All outstanding principal, interest, fees and penalties are written off |
| FR-WO-02 | **Undo write-off** (the latest one, if no recoveries posted after it) |
| FR-WO-03 | **Recovery repayment** on a 601 loan: amount, value date, payment type (incl. `INSURANCE`), external ref. Income goes to INCOME_FROM_RECOVERY. Idempotent like repayments |
| FR-WO-04 | **Charge-off** (status stays 300, flag `charged_off`), with GL reclassification per Fineract's charge-off accounts. Optional in MVP (enabled per tenant) [A] |
| FR-WO-05 | **Provisioning criteria**: categories (STANDARD, SUB-STANDARD, DOUBTFUL, LOSS) × DPD ranges × % per product group, with the provision expense and allowance GL accounts |
| FR-WO-06 | **Provisioning run** (monthly): compute the required allowance per loan (outstanding principal × category %). Post the **delta** versus the previous run as one journal entry per category/product. Maker-checker to post |
| FR-WO-07 | Reports: write-off register, recovery register, provisioning schedule |
| FR-WO-08 | Bulk write-off: `POST /v1/write-offs/batch` (up to 500 loans, per-item results) for the portfolio clean-up during migration |

## 3. Features

| Feature | MVP? |
|---|---|
| Write-off, undo | Yes |
| Recovery repayment (incl. insurance) | Yes |
| Provisioning criteria + monthly run | Yes |
| Charge-off / undo charge-off | Optional (tenant flag) |
| Fraud classification | No |
| Partial write-off | No (Fineract also writes off the full outstanding) |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-WO-01 | Write-off is allowed only on ACTIVE loans with outstanding > 0. The write-off date is ≥ the last transaction value date and ≤ the business date |
| BR-WO-02 | After write-off, standard repayments are rejected (or auto-routed to recovery when `route_repayments_to_recovery=true`; recommended because veend-tenant's collection jobs keep running) |
| BR-WO-03 | Undo write-off is allowed only if no recovery came after it, and is maker-checker |
| BR-WO-04 | Provisioning category by DPD at the run date (default ranges: STANDARD 0–30, SUB-STANDARD 31–90, DOUBTFUL 91–180, LOSS 181+) [A: align with CBN prudential guidelines and finance policy] |
| BR-WO-05 | Written-off loans carry 0 provision (already derecognised) |
| BR-WO-06 | A provisioning run is idempotent per run date. Re-running replaces the unposted draft; a posted run can only be reversed by the next run's delta |

## 5. Data model

```sql
-- write-off and recovery use loan_transaction types WRITEOFF(6) and RECOVERY_REPAYMENT(8)
ALTER TABLE loan ADD COLUMN written_off_on DATE, ADD COLUMN writeoff_reason TEXT, ADD COLUMN charged_off BOOLEAN DEFAULT false;
CREATE TABLE provision_category (id SMALLSERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL);
CREATE TABLE provisioning_criteria (id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL);
CREATE TABLE provisioning_criteria_definition (criteria_id BIGINT REFERENCES provisioning_criteria(id), category_id SMALLINT REFERENCES provision_category(id),
  min_days INT NOT NULL, max_days INT NOT NULL, provision_pct NUMERIC(9,6) NOT NULL,
  liability_account_id BIGINT REFERENCES gl_account(id), expense_account_id BIGINT REFERENCES gl_account(id),
  PRIMARY KEY (criteria_id, category_id));
CREATE TABLE provisioning_criteria_product (criteria_id BIGINT REFERENCES provisioning_criteria(id), product_id BIGINT REFERENCES loan_product(id), PRIMARY KEY (product_id));
CREATE TABLE provisioning_entry (id BIGSERIAL PRIMARY KEY, run_date DATE UNIQUE NOT NULL, status TEXT NOT NULL CHECK (status IN ('DRAFT','POSTED')),
  journal_entry_id BIGINT, command_id BIGINT);
CREATE TABLE provisioning_entry_line (entry_id BIGINT REFERENCES provisioning_entry(id), product_id BIGINT, category_id SMALLINT, office_id BIGINT,
  outstanding NUMERIC(19,6), required NUMERIC(19,6), previous NUMERIC(19,6), delta NUMERIC(19,6), PRIMARY KEY (entry_id, product_id, category_id, office_id));
```

## 6. API endpoints

- `POST /v1/loans/{id}/write-off` (`{date, reason, note}`), `DELETE /v1/loans/{id}/write-off`
- `POST /v1/loans/{id}/recoveries` (`{amount, valueDate, paymentType, externalRef}`)
- `POST /v1/write-offs/batch`
- `POST /v1/loans/{id}/charge-off`, `DELETE /v1/loans/{id}/charge-off`
- `GET/POST /v1/provisioning-criteria`
- `POST /v1/provisioning-entries` (`{runDate}`, creates a DRAFT), `POST /v1/provisioning-entries/{id}/post`, `GET /v1/provisioning-entries/{id}`
- Compat: `POST /loans/{id}/transactions?command=writeoff` and `?command=recoverypayment`, for veend-tenant change M3 when it moves from local flags to CBA write-offs.

## 7. Service-layer responsibilities

Commands `WriteOffLoan`, `UndoWriteOff`, `RecoveryRepayment`, `ChargeOff`, `UndoChargeOff`, `CreateProvisioningEntry`, `PostProvisioningEntry`. `ProvisioningCalculator` (pure).

## 8. Dependencies

loan-lifecycle, loan-transactions, cob-jobs (DPD), accounting.

## 9. Integration

Events `loan.written_off`, `loan.writeoff_undone`, `loan.recovery_posted`, `provisioning.posted`. veend-tenant's `writeOffLoan` job becomes: call the CBA write-off, then set the local flag on success (M3).

## 10. Authorization

`WRITEOFF_LOAN` (+ `_CHECKER`, mandatory), `UNDOWRITEOFF_LOAN` (+ `_CHECKER`), `RECOVERYPAYMENT_LOAN`, `CHARGEOFF_LOAN` (+ `_CHECKER`), `CREATE_PROVISIONING`, `POST_PROVISIONING` (+ `_CHECKER`).

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Write-off (accrual) | LOSSES_WRITTEN_OFF (principal + accrued interest/fees/penalties) | LOAN_PORTFOLIO, INTEREST_/FEES_/PENALTIES_RECEIVABLE |
| Write-off (accrual-upfront, unearned interest) | INTEREST_ON_LOANS (reverse unearned portion) [A finance policy] | INTEREST_RECEIVABLE |
| Recovery | Clearing (payment type, e.g. INSURANCE clearing) | INCOME_FROM_RECOVERY |
| Provision increase | PROVISION_EXPENSE | PROVISION_ALLOWANCE (contra-asset) |
| Provision release | PROVISION_ALLOWANCE | PROVISION_EXPENSE (or recovery income) |

If the loan was provisioned, finance may prefer write-off against the allowance (Dr PROVISION_ALLOWANCE) rather than straight to expense. **[A] Policy decision.** It is configurable: `writeoff_against_allowance` (bool).

## 12. Edge cases

- Write-off on a loan with an unposted overpayment: invalid, since an overpaid loan has no outstanding.
- Collections arriving through veend-tenant's normal repayment pipeline after write-off: routed to recovery when the flag is on. The compat `repayment` on a 601 loan returns 200 with a recovery transaction.
- Migration: loans flagged `isWriteOff` in veend-tenant but ACTIVE in LendLot are **written off at migration** with value date `writeOffDate`, after finance approval (D6 §5 step T6).

## 13. Testing

Golden: write-off and recovery journals versus Fineract. Provisioning delta property (Σ deltas over runs = current required). Bulk write-off idempotency.

## 14. Status & gaps

- Not implemented. veend-tenant uses local flags only ([V] DF-04).
- [A] Finance policies: unearned-interest treatment under upfront accrual; write-off against allowance; provisioning percentages; charge-off usage.
