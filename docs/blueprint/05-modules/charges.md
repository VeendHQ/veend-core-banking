# Module: Charges (fees and penalties)

> Phase: P2 (definitions, schedule-time fees), P4 (overdue penalties) · Priority: P0 · Status: Spec draft
> Fineract reference: `fineract-charge` (`Charge`, `ChargeTimeType`, `ChargeCalculationType`), `m_charge`, `m_loan_charge`, `m_loan_installment_charge`, COB `ApplyChargeToOverdueLoans`
> veend-tenant touchpoints: `createCharge` / `updateCharge` / `deleteCharge` (`Lendlotcharges` controller), product charges, `deleteLoanCharge` (dead). Defaults are `chargeTimeType: 8`, `chargeCalculationType: 2`, `chargeAppliesTo: 1`, `chargePaymentMode: 0` ([V] `lendlot/index.js:1098-1114`)

## 1. Overview & business purpose

This module defines fees and penalties, attaches them to products and loans, computes their amounts, and tracks due, paid, waived and written-off amounts per loan and installment.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-CHG-01 | CRUD for charge definitions: `name`, `appliesTo` (LOAN), `currency`, `amount` (flat amount or percentage), `timeType`, `calculationType`, `paymentMode` (REGULAR only for MVP), `penalty` flag, `minCap`, `maxCap`, `feeInterval` + `feeFrequency` (for overdue), `active` |
| FR-CHG-02 | Supported **time types** (Fineract ids, so compat matches): `DISBURSEMENT(1)`, `SPECIFIED_DUE_DATE(2)`, `INSTALMENT_FEE(8)`, `OVERDUE_INSTALLMENT(9)`. Others are rejected with 400 in MVP |
| FR-CHG-03 | Supported **calculation types**: `FLAT(1)`, `PERCENT_OF_AMOUNT(2)`, `PERCENT_OF_AMOUNT_AND_INTEREST(3)`, `PERCENT_OF_INTEREST(4)`, `PERCENT_OF_DISBURSEMENT_AMOUNT(5)` |
| FR-CHG-04 | Attach charges to a loan at creation (from the product, with an amount override if the product allows) or later (specified-due-date fees, manual penalties) |
| FR-CHG-05 | Waive a loan charge (whole or one installment's portion); undo a waiver |
| FR-CHG-06 | Remove a loan charge before disbursement |
| FR-CHG-07 | COB applies overdue penalties per BR-CHG-05 |
| FR-CHG-08 | Expose net disbursement: Σ disbursement-time fees marked "deduct from disbursement" (see BR-CHG-03) |

## 3. Features

| Feature | MVP? |
|---|---|
| Disbursement fee (flat or %) | Yes: the upfront fee is used in production ([V] `feeChargesAtDisbursementCharged`, 72 reads) |
| Instalment fee | Yes: the default type in `createCharge` |
| Specified-due-date fee | Yes |
| Overdue penalty (one-off or recurring with interval) | Yes |
| Tranche, savings, share and client charges | No |
| Tax on charges | No ([A] finance) |
| Charge payment by account transfer | No |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-CHG-01 | **Amount computation** (matches [V] `AbstractCumulativeLoanScheduleGenerator.calculateInstallmentCharge` `:2130`): for INSTALMENT_FEE, `PERCENT_OF_AMOUNT` uses the installment's **principal**; `…_AND_INTEREST` uses principal + interest; `PERCENT_OF_INTEREST` uses interest; FLAT uses a fixed amount per installment. For DISBURSEMENT, % is applied to the approved/disbursed principal (calc 2 or 5) |
| BR-CHG-02 | `minCap` and `maxCap` apply to percentage charges after computation, then rounding is applied using currency digits and the product rounding mode |
| BR-CHG-03 | **Disbursement-time fees are collected at disbursement** as `REPAYMENT_AT_DISBURSEMENT(5)` (Fineract semantics). Net cash to the borrower = principal − Σ such fees. veend-tenant already computes `disbursementAmount = totalPrincipalDisbursed − feeChargesAtDisbursementCharged` ([V] `Users.js:2273`). Fineract-Mod returns both values explicitly |
| BR-CHG-04 | A penalty charge (`penalty=true`) is allocated under the product strategy as PENALTY. A non-penalty is a FEE |
| BR-CHG-05 | **Overdue penalty**: for each installment overdue more than `graceOnArrearsAgeing` days, at COB on (due date + grace + 1), add a charge instance. If `feeInterval/feeFrequency` is set, repeat every interval while the installment stays overdue, up to `maxOccurrences` [A]. % penalties use the overdue principal (calc 2), principal + interest (3), or interest (4) of that installment |
| BR-CHG-06 | Changing a charge definition affects only loans created after the change. Existing loans keep the snapshot in `loan_charge` |
| BR-CHG-07 | A charge in use by an active product cannot be deleted (the compat `DELETE /charges/{id}` returns a Fineract-style 403 error). It can be deactivated |

## 5. Data model

```sql
CREATE TABLE charge (id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL, applies_to SMALLINT NOT NULL DEFAULT 1,
  currency_code CHAR(3) NOT NULL, amount NUMERIC(19,6) NOT NULL, time_type SMALLINT NOT NULL, calculation_type SMALLINT NOT NULL,
  payment_mode SMALLINT NOT NULL DEFAULT 0, is_penalty BOOLEAN NOT NULL DEFAULT false, active BOOLEAN NOT NULL DEFAULT true,
  min_cap NUMERIC(19,6), max_cap NUMERIC(19,6), fee_interval SMALLINT, fee_frequency SMALLINT,
  legacy_fineract_id BIGINT UNIQUE, deleted BOOLEAN DEFAULT false);
CREATE TABLE loan_charge (id BIGSERIAL PRIMARY KEY, loan_id BIGINT NOT NULL REFERENCES loan(id), charge_id BIGINT NOT NULL REFERENCES charge(id),
  is_penalty BOOLEAN NOT NULL, time_type SMALLINT NOT NULL, calculation_type SMALLINT NOT NULL,
  percentage NUMERIC(9,6), amount NUMERIC(19,6) NOT NULL, due_date DATE,
  amount_paid NUMERIC(19,6) DEFAULT 0, amount_waived NUMERIC(19,6) DEFAULT 0, amount_written_off NUMERIC(19,6) DEFAULT 0,
  amount_outstanding NUMERIC(19,6) NOT NULL, active BOOLEAN DEFAULT true, overdue_installment_id BIGINT, source TEXT);
CREATE TABLE loan_installment_charge (loan_charge_id BIGINT REFERENCES loan_charge(id), installment_id BIGINT REFERENCES loan_installment(id),
  amount NUMERIC(19,6) NOT NULL, amount_paid NUMERIC(19,6) DEFAULT 0, amount_waived NUMERIC(19,6) DEFAULT 0, amount_written_off NUMERIC(19,6) DEFAULT 0,
  PRIMARY KEY (loan_charge_id, installment_id));
```

## 6. API endpoints

- **Native:**
  - `GET/POST /v1/charges`, `PATCH/DELETE /v1/charges/{id}`
  - `POST /v1/loans/{id}/charges`
  - `POST /v1/loans/{id}/charges/{lcId}/waiver` (`{installmentNumber?}`), `DELETE /v1/loans/{id}/charges/{lcId}` (pre-disbursement only)
- **Compat:** `POST /charges` (returns `{resourceId}`), `PUT /charges/{id}` (returns `{resourceId, changes}`), `DELETE /charges/{id}` (returns `{resourceId}`).

## 7. Service-layer responsibilities

`ChargeCalculator` (pure, used by the schedule engine and COB), commands (`CreateCharge`, `UpdateCharge`, `DeleteCharge`, `AddLoanCharge`, `WaiveLoanCharge`, `UndoWaiveLoanCharge`, `ApplyOverduePenalty` from COB).

## 8. Dependencies

Organisation (currency). Consumed by loan-products, the schedule engine, loan-lifecycle, loan-transactions (allocation) and COB.

## 9. Integration

Events `loan.charge_applied` and `loan.charge_waived`.

## 10. Authorization

`CREATE/UPDATE/DELETE/READ_CHARGE`, `CREATE_LOANCHARGE`, `WAIVE_LOANCHARGE` (maker-checker above a threshold), `DELETE_LOANCHARGE`.

## 11. Transactional & accounting implications

| Event | Accrual rules | Cash rule |
|---|---|---|
| Fee at disbursement (collected) | Dr FUND_SOURCE/clearing, Cr INCOME_FROM_FEES | same |
| Instalment fee (ACCRUAL_UPFRONT) | At disbursement: Dr FEES_RECEIVABLE, Cr INCOME_FROM_FEES (all installments) | — |
| Instalment fee (ACCRUAL_PERIODIC) | On due date (COB): Dr FEES_RECEIVABLE, Cr INCOME_FROM_FEES | — |
| Penalty applied | Dr PENALTIES_RECEIVABLE, Cr INCOME_FROM_PENALTIES (accrual) | none until paid |
| Waiver | Dr INCOME_FROM_FEES/PENALTIES, Cr *_RECEIVABLE (accrual) | none |

## 12. Edge cases

- Percentage fee with rounding remainder across installments: Fineract computes per installment independently. This must be matched exactly (golden).
- A waiver on a partly paid installment charge waives only the outstanding part.
- Penalty applied, then the triggering repayment is backdated before the due date: the replay (see [loan-transactions](loan-transactions.md)) **removes** penalties whose trigger no longer holds and posts an ACCRUAL_ADJUSTMENT.

## 13. Testing

- Calculator unit tests for every (time type × calc type) combination including caps.
- Golden fixtures from Fineract for products with a 2% instalment fee and an upfront flat or % fee.
- COB penalty idempotency test: the same COB date twice produces one penalty.

## 14. Status & gaps

- Not implemented.
- [A] Production charge catalogue (`m_charge`) and whether any overdue penalties are configured today. veend-tenant never applies penalties explicitly ([V]), so if LendLot has no penalty charges, MVP can ship with penalties disabled per tenant.
