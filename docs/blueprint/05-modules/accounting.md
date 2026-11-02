# Module: Accounting (chart of accounts, journal entries, product mapping, closures)

> Phase: P3 · Priority: P0 · Status: Spec draft
> Fineract reference: `fineract-accounting` (`JournalEntry`, `ProductToGLAccountMapping`, `GLClosure`, `AccountingProcessorHelper`), `GLAccount` / `GLAccountType` ([V] `:25-29`), `AccountingRuleType` ([V] `:30-33`), `AccrualBasedAccountingProcessorForLoan`, `CashBasedAccountingProcessorForLoan`, `AccountingConstants` (roles, [V] `:37-166`)
> veend-tenant touchpoints: indirect. Hard-coded GL ids in `createLoanProduct` ([V] `lendlot/index.js:870-881`). No GL reads today.

## 1. Overview & business purpose

This module is the tenant's general ledger: the chart of accounts, immutable balanced journal entries (automatic from lending events, plus manual), role-based mapping from product events to accounts, period closures, running balances and the trial balance. It is the authoritative financial record for the lender's books.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-ACC-01 | CoA: CRUD for GL accounts with `glCode` (unique), name, type (ASSET/LIABILITY/EQUITY/INCOME/EXPENSE), usage (HEADER/DETAIL), parent, `manualEntriesAllowed`, `disabled`, and a tag (classification) |
| FR-ACC-02 | Seed the CoA from a **template** at tenant provisioning (default Veend lending template = the LendLot CoA structure, ids remapped) |
| FR-ACC-03 | `PostingService.post(PostingRequest[])`: resolve roles to accounts (product mapping, optional payment-type or charge override), validate balance and closures, persist the entry and lines, update `gl_balance` |
| FR-ACC-04 | Manual journal entries with ≥ 2 lines, balanced, DETAIL accounts with `manualEntriesAllowed`, an optional reference and attachments. Maker-checker mandatory |
| FR-ACC-05 | Reversal of a journal entry (manual entries only; system entries are reversed through their source transaction) |
| FR-ACC-06 | GL closures per office: `closingDate`. Posting at or before it is rejected. Delete the latest closure (reopen) with maker-checker |
| FR-ACC-07 | Queries: journal entries by account, office, date range, source, loan or transaction; account ledger with running balance; trial balance as of a date; balance sheet and income statement (by account type and tag) |
| FR-ACC-08 | Financial-activity mappings (Fineract `acc_gl_financial_activity_account`): e.g. ASSET_TRANSFER, LIABILITY_TRANSFER, OPENING_BALANCES_CONTRA |
| FR-ACC-09 | **Opening balances** posting for migration (source type `MIGRATION`) |

## 3. Features

| Feature | MVP? |
|---|---|
| CoA + template seeding | Yes |
| Automatic postings: loan events (disburse, accrual, repayment, waiver, write-off, recovery, refund, reversal) | Yes |
| Accounting rules NONE / CASH / ACCRUAL_UPFRONT / ACCRUAL_PERIODIC | Yes |
| Payment-type clearing accounts | Yes |
| Manual JEs + maker-checker | Yes |
| Closures | Yes |
| Provisioning postings | Yes (see [writeoff-provisioning](writeoff-provisioning.md)) |
| Accounting rules as JE templates (Fineract `acc_accounting_rule`) | No |
| Multi-currency revaluation | No |
| Accruals for non-loan items | No |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-ACC-01 | Every entry is balanced: Σ debit = Σ credit, per currency (deferred constraint) |
| BR-ACC-02 | Lines only on DETAIL, non-disabled accounts |
| BR-ACC-03 | Entries are immutable. Corrections happen by reversal (`reverses_entry_id`). A reversal is dated the original date if the period is open; otherwise the first open date with the original value date recorded |
| BR-ACC-04 | **Role resolution order:** (product, role, paymentType, charge) → (product, role, paymentType) → (product, role, charge) → (product, role). A missing mapping is a configuration error that fails the command (422 `GL_MAPPING_MISSING`), so it is never posted to suspense silently |
| BR-ACC-05 | Account type sign conventions: ASSET and EXPENSE are debit-normal; LIABILITY, EQUITY and INCOME are credit-normal. `gl_balance.balance` is stored debit-positive |
| BR-ACC-06 | A GL account with a non-zero balance, or used in mappings, cannot be disabled or deleted |
| BR-ACC-07 | Closures are monotonic per office. Only the latest can be deleted |
| BR-ACC-08 | Trial balance invariant: Σ all `gl_balance` = 0 per tenant and currency. Checked after COB and alarmed (operations §1) |

## 5. Data model

```sql
CREATE TABLE gl_account (id BIGSERIAL PRIMARY KEY, gl_code TEXT UNIQUE NOT NULL, name TEXT NOT NULL,
  type SMALLINT NOT NULL CHECK (type BETWEEN 1 AND 5), usage SMALLINT NOT NULL CHECK (usage IN (1,2)), -- 1 DETAIL, 2 HEADER (Fineract)
  parent_id BIGINT REFERENCES gl_account(id), hierarchy TEXT NOT NULL, manual_entries_allowed BOOLEAN DEFAULT true,
  disabled BOOLEAN DEFAULT false, tag TEXT, description TEXT, legacy_fineract_id BIGINT UNIQUE);
CREATE TABLE gl_balance (gl_account_id BIGINT REFERENCES gl_account(id), office_id BIGINT, currency_code CHAR(3), stripe SMALLINT DEFAULT 0,
  balance NUMERIC(19,6) NOT NULL DEFAULT 0, PRIMARY KEY (gl_account_id, office_id, currency_code, stripe));
CREATE TABLE gl_closure (id BIGSERIAL PRIMARY KEY, office_id BIGINT NOT NULL, closing_date DATE NOT NULL, comments TEXT,
  command_id BIGINT NOT NULL, deleted BOOLEAN DEFAULT false, UNIQUE (office_id, closing_date));
CREATE TABLE financial_activity_mapping (activity SMALLINT PRIMARY KEY, gl_account_id BIGINT NOT NULL REFERENCES gl_account(id));
CREATE TABLE posting_rule (accounting_rule TEXT, event TEXT, part TEXT, debit_role TEXT, credit_role TEXT, version INT,
  PRIMARY KEY (accounting_rule, event, part, version));   -- seeded from the table in processing.md §3
-- journal_entry / journal_line: see data-architecture.md §3
CREATE INDEX ix_jl_account ON journal_line (gl_account_id, entry_id);
CREATE INDEX ix_je_date ON journal_entry (entry_date, office_id);
CREATE INDEX ix_je_source ON journal_entry (source_type, source_id);
```

`PostingRequest` (TypeScript):

```ts
interface PostingRequest {
  event: 'DISBURSEMENT'|'FEE_AT_DISBURSEMENT'|'ACCRUAL'|'ACCRUAL_ADJUSTMENT'|'REPAYMENT'|'WAIVE_INTEREST'|'WAIVE_CHARGE'
       |'WRITEOFF'|'RECOVERY'|'REFUND'|'PROVISION'|'MIGRATION_OPENING';
  accountingRule: 'CASH'|'ACCRUAL_UPFRONT'|'ACCRUAL_PERIODIC';
  productId: number; officeId: number; currency: string; valueDate: LocalDate;
  paymentTypeId?: number; loanId?: number; clientId?: number;
  parts: { part: 'PRINCIPAL'|'INTEREST'|'FEE'|'PENALTY'|'OVERPAYMENT'|'TOTAL'; amount: Decimal; chargeId?: number }[];
  source: { type: string; id: number }; commandId: number;
}
```

## 6. API endpoints

- `GET/POST /v1/gl-accounts`, `PATCH /v1/gl-accounts/{id}`, `GET /v1/gl-accounts/{id}/ledger?from&to`
- `GET /v1/journal-entries?account&office&from&to&loanId&sourceType&sourceId`
- `POST /v1/journal-entries` (manual), `POST /v1/journal-entries/{id}/reversal`
- `GET/POST /v1/gl-closures`, `DELETE /v1/gl-closures/{id}`
- `GET /v1/trial-balance?asOf&office`
- `GET/PUT /v1/financial-activity-mappings`
- `GET /v1/posting-rules` (read-only catalogue)
- Compat: none required ([V] veend-tenant never calls `/journalentries`).

## 7. Service-layer responsibilities

`PostingService` (the only writer of ledger tables), `GlMappingResolver`, `ClosureGuard`, `BalanceUpdater` (with striping), `TrialBalanceQuery`, `LedgerQuery`, `ManualJournalCommand`, `CoaTemplateSeeder`.

## 8. Dependencies

platform-core, organisation (office, payment types). Consumed by all financial modules. Has no dependency on lending (ADR-004).

## 9. Integration

- Event `journal.posted` (finance subscribers).
- Export to the ERP: CSV/JSON daily GL extract (P5) [A: target ERP?].

## 10. Authorization

`CREATE/UPDATE/READ_GLACCOUNT`, `CREATE_JOURNALENTRY` (+ `_CHECKER`), `REVERSE_JOURNALENTRY` (+ `_CHECKER`), `CREATE/DELETE_GLCLOSURE` (+ `_CHECKER`), `READ_JOURNALENTRY`, `READ_TRIALBALANCE`.

## 11. Transactional & accounting implications

This module *is* the accounting implication. The posting matrix is in [processing §3](../04-target-architecture/processing.md#3-accounting-engine).

## 12. Edge cases

| Case | Behaviour |
|---|---|
| Hot account contention (FUND_SOURCE) | Striped `gl_balance` rows (stripe = txn id mod 8), summed on read |
| Posting into a closed period from a backdated repayment | 423, unless the tenant has `post_to_first_open_date=true` |
| Migrated unbalanced Fineract groups | Quarantined to `migration_exception` and reported. They are posted against a `MIGRATION_SUSPENSE` account only with finance approval |
| A product mapping changed mid-life | Applies to new entries only. The reversal of an old transaction uses the **original entry's accounts** (mirror), not the current mapping |

## 13. Testing

- Unit: role resolution precedence; sign conventions.
- Property: random sequences of commands keep the trial balance at zero, and Σ portfolio lines per loan = loan principal outstanding.
- Golden: journal lines for each event match Fineract's `acc_gl_journal_entry` for the same scenario (accounts mapped by role).
- Migration: opening trial balance equals the LendLot trial balance on the cut-over date, to the kobo.

## 14. Status & gaps

- Not implemented.
- [A] Production CoA export; mapping of GL ids 5, 9, 30, 32, 31, 24, 6, 10, 7, 8, 19, 33 to names and codes (`acc_gl_account`).
- [A] Finance's ERP and reporting requirements (CBN returns).
