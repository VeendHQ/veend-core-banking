# Module (future): Recurring Deposits and Fixed-Deposit Extensions

> Phase: P7+ · Priority: P3 · Status: Outline spec
> **Fixed deposits are now MVP.** See [../fixed-deposits.md](../fixed-deposits.md). This file covers only what stays in the future.
> Fineract reference: `fineract-savings` (`RecurringDepositProduct`, `DepositProductRecurringDetail`; [V] `fineract-savings/.../savings/domain/`)
> veend-tenant / Giro touchpoints: none

## 1. Overview & business purpose
- **Recurring deposits:** scheduled periodic deposits, e.g. a monthly investment plan.
- **FD extensions:** compounding interest, auto-rollover at maturity, interest-rate charts by amount slab, and FD-backed loans (lien as collateral).

## 2. Functional requirements
- FR-RD-01: RD product (installment amount, frequency, tenure, rate chart).
- FR-RD-02: missed-installment handling (penalty or recalculation).
- FR-RD-03: FD auto-rollover instruction (principal only, or principal + interest).
- FR-RD-04: compounding period for FDs.
- FR-RD-05: FD as loan collateral (hold).

## 3. Features & sub-features
RD products and accounts · RD schedule · rollover · compounding · collateral holds.

## 4. Business rules & validations
An RD installment is due on schedule; a missed installment reduces the rate or triggers a penalty per product. Rollover creates a new FD at maturity using the current product rate.

## 5. Data model
Extends the [fixed-deposits](../fixed-deposits.md) tables: `recurring_deposit_schedule`, `deposit_rollover_instruction`, `deposit_hold`.

## 6. API endpoints
`/v1/recurring-deposit-products`, `/v1/recurring-deposit-accounts`, `/v1/deposit-accounts/{id}/rollover-instruction`, `/v1/deposit-accounts/{id}/holds`.

## 7. Service-layer responsibilities
RD schedule generator; rollover step in COB; compounding in `DepositInterestCalculator`.

## 8. Dependencies on other modules
fixed-deposits, accounting, COB, loan-lifecycle (collateral).

## 9. Integration requirements
Giro for collection of recurring installments (direct debit or wallet sweep).

## 10. Authorization & permissions
`*_RECURRINGDEPOSITACCOUNT`, `ROLLOVER_DEPOSITACCOUNT`, `HOLD_DEPOSITACCOUNT`.

## 11. Transactional & accounting implications
Same roles as fixed deposits. Compounding capitalises interest: Dr INTEREST_PAYABLE / Cr SAVINGS_CONTROL.

## 12. Error handling & edge cases
Missed RD installments; rollover when the product is inactive; a hold larger than the balance.

## 13. Testing requirements
Fineract RD integration scenarios as golden fixtures.

## 14. Implementation status & outstanding gaps
Not planned. [A] Business need.
