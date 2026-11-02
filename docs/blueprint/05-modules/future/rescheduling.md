# Module (future): Loan Restructuring and Rescheduling

> Phase: P7 · Priority: P2 (first post-MVP candidate) · Status: Outline spec
> Fineract reference: `m_loan_reschedule_request`, `m_loan_term_variations`, `LoanRescheduleRequest*`, `reAge`/`reAmortize` transactions (types 29/30, [V] `LoanTransactionType.java:67-68`), status `CLOSED_RESCHEDULE_OUTSTANDING_AMOUNT(602)`
> veend-tenant touchpoints: mandate "restructuring" only (`Loans.restructuredRemitaMandateReferences`; [V] `models/Loans.js`). No CBA restructuring today.

## 1. Overview & business purpose
Change a loan's repayment terms after disbursement (extend the tenor, change the rate, add a payment holiday, re-age arrears) for distressed borrowers, with proper accounting and regulatory classification (restructured loans are tagged).

## 2. Functional requirements
- FR-RS-01: reschedule request (from installment N: new due date, extra terms, new rate, grace), approval (maker-checker), then schedule regeneration.
- FR-RS-02: re-age (move overdue amounts to the end).
- FR-RS-03: re-amortise (spread outstanding over the remaining installments).
- FR-RS-04: a restructure flag and history, for reporting and provisioning (restructured loans may keep their category for N months [A policy]).

## 3. Features & sub-features
Reschedule requests · re-age · re-amortise · term variations · restructure classification.

## 4. Business rules & validations
Only ACTIVE loans. The reschedule start date is ≥ the business date. No reschedule within a closed period. Interest on the extended tenor follows the product method.

## 5. Data model
`loan_reschedule_request`, `loan_term_variation`, `loan_restructure_history`. `loan_installment` versions are kept in `loan_installment_history`.

## 6. API endpoints
`POST /v1/loans/{id}/reschedule-requests`, `/approve`, `/reject`; `POST /v1/loans/{id}/re-age`; `POST /v1/loans/{id}/re-amortize`.

## 7. Service-layer responsibilities
`RescheduleService` (uses the schedule engine with term variations); replay of transactions after regeneration.

## 8. Dependencies on other modules
schedule engine, loan-lifecycle, loan-transactions, COB (classification), provisioning.

## 9. Integration requirements
Event `loan.rescheduled`. veend-tenant must recreate collection mandates on that event (it does this manually today).

## 10. Authorization & permissions
`RESCHEDULE_LOAN` (+ `_CHECKER`), `REAGE_LOAN`, `REAMORTIZE_LOAN`.

## 11. Transactional & accounting implications
Under accrual-upfront, the change in total interest is posted as ACCRUAL / ACCRUAL_ADJUSTMENT. Re-age has no GL effect beyond reclassification.

## 12. Error handling & edge cases
Reschedule with a pending backdated transaction; reschedule followed by reversal of a prior repayment.

## 13. Testing requirements
Fineract reschedule integration tests as golden fixtures.

## 14. Implementation status & outstanding gaps
Not in MVP. [A] Ask the business whether restructures happen off-system today and how they are booked.
