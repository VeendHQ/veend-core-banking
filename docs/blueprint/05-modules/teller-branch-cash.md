# Module: Teller, Vault, Cash and Branch Operations

> Phase: **P6-B** · Priority: P0 for BANK tenants with cash operations · Status: Spec draft
> Scope origin: F-11, F-59, F-60 ([register](../03-gap-analysis/recovery-team-feature-register.md)); call-over report (vault ↔ teller movements, entry codes C102/D102) [V]; TB lines "Vault", "Teller Overages" [V]; guide §3.7 branch access and branch-to-HO reconciliation; §10 "Branch vs head-office GL"
> Fineract reference: `fineract-branch` teller and cashier (`Teller`, `Cashier` [V] `fineract-branch/.../organisation/teller/domain/`)

## 1. Overview & business purpose

This module handles branch cash operations:
- vaults and teller tills;
- cash movements between vault and teller and between branches;
- customer cash deposits and withdrawals (cash posting);
- teller end-of-day balancing with overage/shortage handling;
- branch-level access control, transaction ownership, and branch-to-head-office reconciliation (inter-branch accounts).

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-TLR-01 | Define vaults (per branch) and tellers (till GL account, cash limits, assigned user, open/closed state) |
| FR-TLR-02 | Teller session: open (with the opening float from the vault), transact, close (count by denomination), balance |
| FR-TLR-03 | **Cash movements**: vault → teller ("buy cash"), teller → vault ("sell cash"), vault ↔ bank (CBN/correspondent), inter-branch cash transfer. Both legs in one posting, approval per limit |
| FR-TLR-04 | **Cash posting**: customer cash deposit and withdrawal to CASA and loan repayment in cash, with denomination capture optional. Teller limits; above the limit → supervisor approval |
| FR-TLR-05 | **End-of-day**: physical count vs system till balance. A difference posts to TELLER_OVERAGES (liability) or TELLER_SHORTAGES (receivable from staff/expense) with supervisor sign-off |
| FR-TLR-06 | **Branch access**: users are bound to branch(es); transactions record the owning branch; data scope by office hierarchy (FR-PLT office scope) |
| FR-TLR-07 | **Inter-branch accounting**: a transaction where the customer's home branch ≠ the posting branch auto-generates inter-branch entries (DUE_TO/DUE_FROM per branch pair) |
| FR-TLR-08 | **Branch-to-HO reconciliation**: daily, the inter-branch balances net to zero across branches; branch trial balances sum to the consolidated TB |
| FR-TLR-09 | Teller reports: teller cash position, call-over by teller (FR-RPT-10), vault position, denomination summary |

## 3. Features & sub-features

Vaults · tellers and sessions · cash movements · cash posting · EOD balancing · overages/shortages · branch access · inter-branch accounting · branch/HO reconciliation · teller reports.

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-TLR-01 | A teller can transact only in an open session; one open session per teller |
| BR-TLR-02 | A till balance cannot go negative; a till above its max cash triggers a mandatory sell-to-vault prompt |
| BR-TLR-03 | A session cannot close until the count is recorded and any difference is approved and posted |
| BR-TLR-04 | Inter-branch DUE_TO/DUE_FROM must net to zero at EOD (control, exception if not) |
| BR-TLR-05 | A cash posting carries the teller, session, branch and terminal/IP (feeds the call-over) |

## 5. Data model

```sql
CREATE TABLE vault (id BIGSERIAL PRIMARY KEY, office_id BIGINT NOT NULL, gl_account_id BIGINT NOT NULL, name TEXT);
CREATE TABLE teller (id BIGSERIAL PRIMARY KEY, office_id BIGINT NOT NULL, gl_account_id BIGINT NOT NULL, name TEXT, max_cash NUMERIC(19,6), txn_limit NUMERIC(19,6), status TEXT);
CREATE TABLE teller_assignment (teller_id BIGINT, user_id BIGINT, from_date DATE, to_date DATE, PRIMARY KEY (teller_id, user_id, from_date));
CREATE TABLE teller_session (id BIGSERIAL PRIMARY KEY, teller_id BIGINT, user_id BIGINT, business_date DATE, opened_at TIMESTAMPTZ, closed_at TIMESTAMPTZ,
  opening_balance NUMERIC(19,6), system_closing NUMERIC(19,6), counted_closing NUMERIC(19,6), difference NUMERIC(19,6), denominations JSONB, approved_by TEXT);
CREATE TABLE cash_movement (id BIGSERIAL PRIMARY KEY, type TEXT, from_ref TEXT, to_ref TEXT, amount NUMERIC(19,6), journal_entry_id BIGINT, command_id BIGINT);
CREATE TABLE interbranch_account (office_id BIGINT, counterparty_office_id BIGINT, gl_account_id BIGINT, PRIMARY KEY (office_id, counterparty_office_id));
```

## 6. API endpoints

`GET/POST /v1/vaults`, `GET/POST /v1/tellers`, `POST /v1/tellers/{id}/sessions` (open), `POST /v1/teller-sessions/{id}/close`, `POST /v1/cash-movements`, `POST /v1/teller-sessions/{id}/cash-deposits`, `/cash-withdrawals`, `/loan-repayments`, `GET /v1/tellers/{id}/position`, `GET /v1/branch-reconciliation?date`.

## 7. Service-layer responsibilities

`TellerSessionService`, `CashMovementService`, `CashPostingService` (delegates to CASA/loan commands with the till as counter-leg), `InterBranchPoster`, `BranchReconciliationControl` (registered in reconciliation-exceptions).

## 8. Dependencies

organisation (offices), platform-core (users, limits, approvals), savings-current-accounts, loan-transactions, accounting, reconciliation-exceptions.

## 9. Integration requirements

Optional cash-counter devices; receipt printing [A].

## 10. Authorization & permissions

`OPEN_TELLERSESSION`, `CASH_DEPOSIT`, `CASH_WITHDRAWAL` (limits), `CASH_MOVEMENT` (+ `_CHECKER`), `APPROVE_TELLERDIFFERENCE`, `MANAGE_TELLERS`, `MANAGE_VAULTS`.

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Vault → teller | TELLER_TILL | VAULT |
| Customer cash deposit | TELLER_TILL | SAVINGS_CONTROL (customer) |
| Customer cash withdrawal | SAVINGS_CONTROL | TELLER_TILL |
| Overage | TELLER_TILL | TELLER_OVERAGES |
| Shortage | TELLER_SHORTAGES | TELLER_TILL |
| Inter-branch (posting branch B for a home-branch A customer) | B: TELLER_TILL / Cr DUE_TO_A · A: DUE_FROM_B / Cr SAVINGS_CONTROL | |

## 12. Error handling & edge cases

Session left open overnight → auto-lock and a supervisor force-close. Reversal of a cash transaction after session close → posted in the next session with a link. Branch closure → till sweep to vault.

## 13. Testing requirements

EOD balancing scenarios; inter-branch netting property (always zero); call-over completeness (every cash posting appears).

## 14. Implementation status & outstanding gaps

Not implemented. [A] Number of Pettysave branches and tellers; cash limits policy.
