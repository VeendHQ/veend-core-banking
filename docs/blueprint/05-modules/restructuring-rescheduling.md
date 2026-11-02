# Module: Loan Restructuring, Rescheduling and Refinancing

> Phase: **P7-R** (promoted from future; ADR-010) · Priority: P1 · Status: Spec draft
> Scope origin: F-28 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.2, §3.3, §12 ("restructuring, rescheduling, early settlement and write-offs … preserving history"); PAR columns Is Restructured, Is Refinanced, Restructured/Refinanced Tenure and Maturation Date [V structure; 5 restructured loans, ₦1.68m]
> Fineract reference: `LoanRescheduleRequest` ([V] `fineract-loan/.../rescheduleloan/domain/`), `m_loan_term_variations`, `reAge`/`reAmortize` transaction types 29/30 ([V] `LoanTransactionType.java:67-68`), status `CLOSED_RESCHEDULE_OUTSTANDING_AMOUNT(602)`
> veend-tenant touchpoints: mandate "restructuring" only (`Loans.restructuredRemitaMandateReferences`; [V])

## 1. Overview & business purpose

This module changes a loan's contract after disbursement while preserving history and classification integrity:
- **reschedule**: new dates, tenor, rate or grace on the same loan;
- **re-age**: move arrears to the end;
- **re-amortise**: spread the outstanding over the remaining installments;
- **refinance**: close the loan from a new loan;
- **early settlement** is covered in [loan-transactions](loan-transactions.md).

Restructured loans are flagged for regulatory treatment.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-RS-01 | Reschedule request: from installment N or a date; changes (new first due date, extra installments, new rate, grace, installment amount); reason; supporting documents |
| FR-RS-02 | Preview: old vs new schedule and the effect on total interest. Approval (maker-checker, mandatory) → apply: regenerate the schedule, keep `loan_installment_history` (version n) |
| FR-RS-03 | Re-age and re-amortise transactions (Fineract types 29, 30) |
| FR-RS-04 | Refinance: new loan disbursed internally to pay off the old one (status 602 or 600 on the old loan), linked; refinance flag and dates |
| FR-RS-05 | Restructure flag/history: `isRestructured`, count, dates, original maturity, restructured tenure and maturity (as the PAR report shows) |
| FR-RS-06 | Classification policy for restructured loans: keep the pre-restructure classification for a cure period (N on-time payments) before upgrading [A: CBN/finance policy] |
| FR-RS-07 | Events `loan.rescheduled`, `loan.refinanced`, which consumers (veend-tenant mandates) use to recreate collections |

## 3. Features & sub-features

Requests · preview · approval · schedule versioning · re-age · re-amortise · refinance · flags and history · cure-period classification.

## 4. Business rules & validations

BR-RS-01: only ACTIVE loans; not inside a closed period; no pending backdated transactions. BR-RS-02: a restructure cannot reduce principal (that is a waiver or write-off). BR-RS-03: the cure period blocks automatic upgrade of the classification. BR-RS-04: total interest changes are posted as ACCRUAL/ACCRUAL_ADJUSTMENT (upfront rule) or flow through periodic accrual.

## 5. Data model

```sql
CREATE TABLE loan_reschedule_request (id BIGSERIAL PRIMARY KEY, loan_id BIGINT, type TEXT, from_installment INT, from_date DATE, changes JSONB,
  reason TEXT, status TEXT, preview JSONB, command_id BIGINT, applied_on DATE);
CREATE TABLE loan_installment_history (loan_id BIGINT, version INT, installments JSONB, superseded_on DATE, PRIMARY KEY (loan_id, version));
ALTER TABLE loan ADD COLUMN restructure_count INT DEFAULT 0, ADD COLUMN original_maturity DATE, ADD COLUMN refinanced_from BIGINT, ADD COLUMN cure_until DATE;
```

## 6. API endpoints

`POST /v1/loans/{id}/reschedule-requests`, `GET …/{rid}/preview`, `POST …/{rid}/approve|reject`, `POST /v1/loans/{id}/re-age`, `POST /v1/loans/{id}/re-amortize`, `POST /v1/loans/{id}/refinance`.

## 7. Service-layer responsibilities

`RescheduleService` (uses the schedule engine with term variations, then the replay engine for later transactions), `RefinanceService` (composite command), classification-cure hook in COB.

## 8. Dependencies

schedule engine, loan-lifecycle, loan-transactions, cob-jobs, writeoff-provisioning.

## 9. Integration requirements

Events to veend-tenant/Giro for mandate recreation.

## 10. Authorization & permissions

`RESCHEDULE_LOAN` (+ `_CHECKER` mandatory), `REAGE_LOAN`, `REAMORTIZE_LOAN`, `REFINANCE_LOAN` (+ `_CHECKER`).

## 11. Transactional & accounting implications

Reschedule: interest delta postings (accrual rules). Refinance: the new loan disburses to an internal clearing account and repays the old loan (net zero cash), with fees per product.

## 12. Error handling & edge cases

Reschedule followed by reversal of a pre-reschedule payment (the replay engine runs against the new schedule version and logs the conflict); a reschedule on a written-off loan is not allowed (undo the write-off first).

## 13. Testing requirements

Golden: Fineract reschedule integration scenarios. History preservation (old schedule retrievable). Cure-period classification time-travel tests.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Restructuring policy (approval authority, cure period), current practice at Pettysave and Veend.
