# ADR-004: Immutable double-entry ledger with role-based posting

- Status: Proposed [R]

## Context

- Fineract stores journal entries as single-sided rows grouped by `transaction_id`, and resolves accounts through product→GL role mappings ([V] `acc_gl_journal_entry`, `ProductToGLAccountMapping`, `AccrualAccountsForLoan`).
- veend-tenant hard-codes GL ids per product ([V] `lendlot/index.js:870-881`).
- Regulators and auditors require that corrections are made by reversal, not edits.

## Decision

1. **Header + lines** (`journal_entry`, `journal_line`). Each line is debit **or** credit, never both. A deferrable constraint trigger enforces Σdebit = Σcredit per entry at commit.
2. **Immutable.** No UPDATE or DELETE grants on ledger tables. Corrections are new entries with `reverses_entry_id`.
3. **Role-based posting.** Lending modules emit `PostingRequest{event, productId, paymentTypeId, officeId, valueDate, parts}`. The ledger maps roles to accounts using `product_gl_mapping`, with an optional payment-type override. Role names are **identical to Fineract's** `AccrualAccountsForLoan` / `CashAccountsForLoan` (FUND_SOURCE, LOAN_PORTFOLIO, INTEREST_ON_LOANS, INCOME_FROM_FEES, INCOME_FROM_PENALTIES, LOSSES_WRITTEN_OFF, INTEREST_RECEIVABLE, FEES_RECEIVABLE, PENALTIES_RECEIVABLE, TRANSFERS_SUSPENSE, OVERPAYMENT, INCOME_FROM_RECOVERY, …), so migrated `acc_product_mapping` maps 1:1.
4. **Entry date = value date** of the source transaction. If that falls in a closed period, the command is rejected (`423`). The tenant can instead opt in to "post at the first open date", with the original value date stored as a line attribute.
5. **Running balances** in `gl_balance(account, office, currency)` are updated in the posting transaction. A daily job verifies them against Σ lines.
6. **Sub-ledger dimension.** `journal_line.loan_id` and `client_id` let the loan portfolio account be reconciled to individual loans.

## Consequences

- The posting rule table (see [processing §3](../processing.md#3-accounting-engine)) becomes a versioned artefact, owned jointly by finance and engineering.
- Reporting queries are simple aggregates over lines.
- Migration converts Fineract single-sided rows into headers + lines grouped by `transaction_id` (D6). Any historically unbalanced group is quarantined and reported.
