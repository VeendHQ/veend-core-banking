# Module (future): Savings Accounts

> Phase: P7+ · Priority: P3 · Status: Outline spec
> Fineract reference: `fineract-savings` (198 Java files; [V] D1.1 §1), `SavingsAccount`, `SavingsProduct`, `SavingsCOBBusinessStep`
> veend-tenant touchpoints: none. Giro (`bill-payment-portal`) opens one Fineract savings account per wallet account (product `CBASavingsProductId.NGN`), places liens on it (`holdAmount`/`releaseAmount`), and has migrated the wallet **balances** to its own SQL ledger ("V3", `Balances.balance DOUBLE`) ([V] [D1.4 §2](../../01-assessment/bill-payment-portal.md#2-what-giro-uses-fineract-for)). Whether the Giro wallet ledger moves into Fineract-Mod is decision D9. Until then, compat serves a wallet-mirror stub. Fixed deposits are covered separately in [../fixed-deposits.md](../fixed-deposits.md).

## 1. Overview & business purpose
Deposit accounts with balances, interest posting and withdrawals. They would only be needed if Veend or a lender takes deposits on the CBA (licence-dependent) or wants loan disbursement to an internal account.

## 2. Functional requirements
- FR-SAV-01: products (nominal rate, compounding, posting period, min balance, overdraft).
- FR-SAV-02: account lifecycle (submit/approve/activate/close).
- FR-SAV-03: deposit, withdrawal, transfer.
- FR-SAV-04: daily interest accrual and periodic posting.
- FR-SAV-05: holds/liens.
- FR-SAV-06: disbursement to savings and repayment from savings (standing instructions).

## 3. Features & sub-features
Savings products · accounts · transactions · interest · charges · holds · standing instructions · dormancy.

## 4. Business rules & validations
Balance ≥ min balance unless overdraft is allowed. Interest uses daily balance × rate / daysInYear, compounded per product. Withdrawals must not breach holds. All money arithmetic follows the platform decimal rules.

## 5. Data model
`savings_product`, `savings_account`, `savings_transaction`, `savings_hold`, `standing_instruction`. Account balance is derived and verified at COB, as with loans.

## 6. API endpoints
`/v1/savings-products`, `/v1/savings-accounts`, `/v1/savings-accounts/{id}/deposits`, `/withdrawals`, `/transfers`, `/holds`.

## 7. Service-layer responsibilities
Commands for each transaction type, a `SavingsInterestCalculator` (pure), and a COB step for accrual and posting.

## 8. Dependencies on other modules
platform-core, clients, accounting, COB, charges.

## 9. Integration requirements
NIBSS/NIP for inbound and outbound transfers (licence-dependent); events `savings.*`.

## 10. Authorization & permissions
`*_SAVINGSACCOUNT`, `DEPOSIT_SAVINGSACCOUNT`, `WITHDRAW_SAVINGSACCOUNT` (with maker-checker thresholds).

## 11. Transactional & accounting implications
Deposits: Dr clearing / Cr SAVINGS_CONTROL (liability). Interest: Dr INTEREST_ON_SAVINGS (expense) / Cr SAVINGS_CONTROL.

## 12. Error handling & edge cases
Concurrent withdrawals (row lock), backdated deposits affecting interest, dormancy reactivation.

## 13. Testing requirements
Golden fixtures from Fineract savings for interest; property tests that balance = Σ transactions.

## 14. Implementation status & outstanding gaps
Not planned for MVP. [A] Business and licensing need not established.
