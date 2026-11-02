# Module: Savings, Current and Commitment-Savings Accounts (incl. overdrafts)

> Phase: **P6-A** (banking track) · Priority: P0 for BANK tenants · Status: Spec draft
> Scope origin: recovery-team features F-17, F-18, F-19, F-33, F-58 ([register](../03-gap-analysis/recovery-team-feature-register.md)); ADR-010
> Fineract reference: `fineract-savings` (`SavingsAccount` 3,801 lines, `SavingsProduct`; overdraft `allowOverdraft`/`overdraftLimit`, `minRequiredBalance`, `lockinPeriodFrequency`, `withHoldTax`; [V]), `RecurringDepositAccount` ([V] `fineract-provider/.../savings/domain/`), `SavingsAccountStatusType` (100–800), accounting roles `AccrualAccountsForSavings` ([V] `AccountingConstants.java:311-326`)
> Current systems: BankOne at Pettysave MFB (BS: Demand ₦38.2m, Voluntary Savings ₦93.3m, Special ₦6.1m, Term ₦85.2m; CBN 201: 2,186 deposit accounts); unnamed-entity TB (Regular Savings, Eazy Savings, Regular Target Savings, Interest-free Savings); Giro wallet mirror ([D1.4 §2](../01-assessment/bill-payment-portal.md#2-what-giro-uses-fineract-for))

## 1. Overview & business purpose

This module covers the deposit accounts of a licensed bank tenant:
- **savings** (voluntary, regular, interest-free);
- **current/demand** accounts, with an optional overdraft;
- **commitment/target savings**: scheduled contributions toward a target, locked until a date or until the target is met, with a penalty on early break.

It keeps balances, interest accrual and posting, withholding tax, holds/liens, dormancy, KYC-tier limits, and overdraft classification and provisioning. Term deposits are in [fixed-deposits](fixed-deposits.md).

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-SAV-01 | Products: type (`SAVINGS`, `CURRENT`, `COMMITMENT`), currency, interest (rate or tiered slabs by balance, compounding and posting period, day count, balance method: daily or minimum monthly), min opening/min balance, withdrawal limits, charges (maintenance, SMS, COT where permitted), overdraft (current only), dormancy rules, lock-in, WHT on interest, GL mapping (savings roles) |
| FR-SAV-02 | Account lifecycle: submit → approve → activate → (dormant ↔ active) → closed. Per-account overrides within product bounds. Joint accounts and mandates (signatories, sole/joint rules) |
| FR-SAV-03 | Transactions: deposit, withdrawal, internal transfer, charge, interest posting, WHT, hold/release, reversal. Every one carries `(channel, externalRef)` uniqueness and value date |
| FR-SAV-04 | **KYC-tier limits** (CBN 3-tier KYC): per-tier max single deposit and max cumulative balance, enforced on credit. Tier from [clients](clients.md) FR-CLI-10 |
| FR-SAV-05 | Daily interest accrual (COB). Posting monthly, quarterly or at period end per product. WHT deducted at posting [A: rate per Nigerian tax rules; 10% currently] |
| FR-SAV-06 | **Holds/liens** with reason and expiry (replaces Giro's Fineract `holdAmount` usage; [V] `Liens.js:423`). Available balance = ledger − holds − min balance + overdraft limit |
| FR-SAV-07 | **Overdrafts** on current accounts: limit, rate, expiry. Daily interest on the debit balance. **Classification of overdrawn accounts** by days continuously overdrawn or beyond the limit, using the CBN bands, which feeds **provision overdrawn accounts** (F-33) |
| FR-SAV-08 | **Unauthorised overdraft detection**: any debit that drives a non-overdraft account negative is rejected by default. If forced (e.g. loan-repayment debit), it creates an exception (see [reconciliation-exceptions](reconciliation-exceptions.md)). This addresses the 737 negative linked accounts seen at Pettysave ([input review I-3](../09-input-reviews/recovery-team-2026-10.md#23-other-inconsistencies)) |
| FR-SAV-09 | **Commitment savings**: contribution schedule (amount, frequency, auto-debit from a source account), target amount/date, early-break penalty (interest forfeiture or %), maturity → transfer to the linked account |
| FR-SAV-10 | Dormancy: inactive N days → dormant (debits blocked, credits allowed); unclaimed-balance escheat per CBN after the prescribed years [A] |
| FR-SAV-11 | Linked accounts: a loan's repayment account (auto-debit on due date via a standing instruction) |
| FR-SAV-12 | Statements, balance enquiry, interest certificate |
| FR-SAV-13 | Account numbers: NUBAN (10-digit with CBN check digit) issued from the tenant's NUBAN range; the internal account number is separate |
| FR-SAV-14 | **Giro wallet mirror** (TREASURY tenant): a thin account type that holds Giro wallet ids and liens for compat calls until/unless the wallet ledger migrates |

## 3. Features & sub-features

Products (savings/current/commitment) · accounts and mandates · deposits/withdrawals/transfers · interest and WHT · holds · overdrafts and their provisioning · commitment schedules · dormancy and escheat · NUBAN · standing instructions · statements.

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-SAV-01 | No debit may reduce the available balance below zero unless overdraft is enabled and within its limit |
| BR-SAV-02 | Credits that breach the KYC tier cap are rejected (`422 KYC_TIER_LIMIT`) |
| BR-SAV-03 | Interest (daily) = `closing balance × rate/100 / dayCountBasis` per tier slab, at full precision. Posted rounded HALF_EVEN with the remainder carried (same rule as BR-FD-04) |
| BR-SAV-04 | Overdraft classification uses the continuous-overdrawn days: 1–30 P&W, 31–60 Substandard, 61–90 Doubtful, ≥ 91 Lost (CBN MFB template; configurable) |
| BR-SAV-05 | A commitment-account early break applies the penalty, posts it, then releases the funds |
| BR-SAV-06 | A dormant account accepts no debit except bank charges [A] |
| BR-SAV-07 | Account closure requires zero balance, no holds and no linked active loans |
| BR-SAV-08 | Every transaction is idempotent on `(channel, externalRef)` |

## 5. Data model

```sql
CREATE TABLE deposit_account_product (id BIGSERIAL PRIMARY KEY, type TEXT CHECK (type IN ('SAVINGS','CURRENT','COMMITMENT','WALLET_MIRROR')),
  name TEXT UNIQUE, current_version INT, legacy_ref TEXT);
CREATE TABLE deposit_account_product_version (product_id BIGINT, version INT, terms JSONB NOT NULL, PRIMARY KEY (product_id, version));
CREATE TABLE casa_account (id BIGSERIAL PRIMARY KEY, nuban CHAR(10) UNIQUE, account_no VARCHAR(20) UNIQUE NOT NULL, client_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL, product_version INT NOT NULL, office_id BIGINT NOT NULL, status SMALLINT NOT NULL,
  ledger_balance NUMERIC(19,6) NOT NULL DEFAULT 0, hold_total NUMERIC(19,6) NOT NULL DEFAULT 0,
  overdraft_limit NUMERIC(19,6) DEFAULT 0, overdraft_rate NUMERIC(9,6), overdraft_expiry DATE,
  interest_accrued_exact NUMERIC(30,12) DEFAULT 0, last_activity_on DATE, dormant_since DATE, overdrawn_since DATE,
  classification TEXT, kyc_tier SMALLINT, legacy_ref TEXT UNIQUE, attributes JSONB DEFAULT '{}', version INT DEFAULT 0);
CREATE TABLE casa_transaction (id BIGSERIAL PRIMARY KEY, account_id BIGINT NOT NULL, type TEXT NOT NULL, value_date DATE, booking_date DATE,
  amount NUMERIC(19,6) NOT NULL, running_balance NUMERIC(19,6) NOT NULL, channel TEXT, external_ref VARCHAR(120), reversed BOOLEAN DEFAULT false,
  command_id BIGINT NOT NULL, UNIQUE (channel, external_ref));
CREATE TABLE casa_hold (id BIGSERIAL PRIMARY KEY, account_id BIGINT, amount NUMERIC(19,6), reason TEXT, external_ref TEXT UNIQUE, expires_on DATE, released_at TIMESTAMPTZ);
CREATE TABLE account_mandate (account_id BIGINT, client_id BIGINT, role TEXT, rule TEXT, PRIMARY KEY (account_id, client_id));
CREATE TABLE commitment_plan (account_id BIGINT PRIMARY KEY, target_amount NUMERIC(19,6), target_date DATE, contribution NUMERIC(19,6),
  frequency TEXT, source_account_id BIGINT, break_penalty JSONB);
CREATE TABLE standing_instruction (id BIGSERIAL PRIMARY KEY, from_account BIGINT, to_type TEXT, to_id BIGINT, amount_rule TEXT, schedule TEXT, active BOOLEAN);
```

## 6. API endpoints

- `GET/POST /v1/casa-products`
- `POST /v1/casa-accounts`, `GET /v1/casa-accounts/{id}`, `/by-nuban/{nuban}`
- `POST /v1/casa-accounts/{id}/deposits`, `/withdrawals`, `/transfers`, `/holds`, `DELETE …/holds/{holdId}`
- `POST /v1/casa-accounts/{id}/overdraft` (set/renew), `/activation`, `/closure`, `/reactivation`
- `GET /v1/casa-accounts/{id}/balance`, `/transactions`, `/statement?from&to`
- `POST /v1/commitment-plans`, `POST /v1/casa-accounts/{id}/early-break`
- `GET/POST /v1/standing-instructions`
- Compat (Giro): `/savingsaccounts/*` wallet-mirror ids and `holdAmount`/`releaseAmount` (D2.7 §4, G-S8/G-S9)

## 7. Service-layer responsibilities

Commands per transaction type; `CasaInterestCalculator` (pure); COB steps `CasaAccrual`, `InterestPosting`, `DormancyStep`, `OverdraftClassification`, `CommitmentContributionStep`, `StandingInstructionStep`; `NubanGenerator`.

## 8. Dependencies

clients (KYC tier), accounting, organisation (branches), COB, charges (deposit charges), payments-transfers-bills (inbound/outbound), teller-branch-cash (cash posting), writeoff-provisioning (overdraft provisioning).

## 9. Integration requirements

NIBSS (account name enquiry against the NUBAN; inbound NIP credits) via [payments-transfers-bills](payments-transfers-bills.md); NDIC (insured-deposit reporting via [regulatory-reporting](regulatory-reporting.md)); SMS/email alerts through events.

## 10. Authorization & permissions

`CREATE/READ/CLOSE_SAVINGSACCOUNT`, `DEPOSIT_*`, `WITHDRAW_*` (limits per role, FR-PLT-11), `HOLDAMOUNT_*`, `APPROVE_OVERDRAFT` (+ `_CHECKER`), `REACTIVATE_DORMANT` (+ `_CHECKER`).

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Deposit (cash/transfer) | Teller till / NIBSS settlement / clearing | SAVINGS_CONTROL (per product: demand, savings, commitment) |
| Withdrawal | SAVINGS_CONTROL | Till / settlement |
| Interest accrual | INTEREST_ON_SAVINGS (expense) | INTEREST_PAYABLE |
| Interest posting | INTEREST_PAYABLE | SAVINGS_CONTROL (net), WHT_PAYABLE (tax) |
| Overdraft drawn | OVERDRAFT_PORTFOLIO_CONTROL (asset) | — (the account balance goes negative; the control account is reclassed daily) |
| Overdraft interest | OVERDRAFT_INTEREST_RECEIVABLE | INCOME_FROM_INTEREST |
| Charges | SAVINGS_CONTROL | INCOME_FROM_FEES |
| Early-break penalty | SAVINGS_CONTROL | INCOME_FROM_PENALTIES |
| Escheat | SAVINGS_CONTROL | ESCHEAT_LIABILITY |

## 12. Error handling & edge cases

Concurrent withdrawals (row lock on the account); backdated deposits changing interest (re-accrual from the value date); a hold that exceeds the balance (allowed; available goes negative only for court/regulatory holds [A]); NIP credit to a dormant account (accepted, reactivation flagged).

## 13. Testing requirements

Golden: Fineract savings interest (daily balance, monthly posting) for equivalent configs. Property: ledger balance = Σ transactions; available-balance invariants. Concurrency on withdrawals. UAT: statement closing = ledger.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Pettysave's BankOne product catalogue and interest/charge settings; CBN KYC tier limits in force; WHT treatment; dormancy/escheat periods; whether Giro's wallet ledger moves here (decision D9).
