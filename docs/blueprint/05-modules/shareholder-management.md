# Module: Shareholder Management

> Phase: **P7-E** · Priority: P2 · Status: Spec draft
> Scope origin: F-56 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.7 "record shareholding, share issuance and dividends, with appropriate approval and accounting entries"; Pettysave BS: Ordinary Shares ₦200m, Deposit for Shares ₦92m [V]; TB: Share Capital, Preference Shares, Share Premium, Deposit for Share, Statutory and Regulatory Risk Reserves, coupon payment expense [V]
> Fineract reference: `ShareProduct`, `ShareAccount` ([V] `fineract-provider/.../portfolio/shareproducts`, `shareaccounts`)

## 1. Overview & business purpose

This module keeps the bank tenant's **register of members/shareholders**: share classes (ordinary, preference), holdings, issuance and allotment (including deposits for shares awaiting allotment), transfers, dividends and preference coupons, and the related reserves postings (statutory reserve, regulatory risk reserve). It supports the CBN capital-adequacy and ownership returns.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-SHR-01 | Share classes: ordinary, preference (coupon rate, cumulative?, redeemable?); nominal value; authorised units |
| FR-SHR-02 | Shareholder register: individual or corporate (linked to a client or a standalone party), KYC, holdings by class, certificates |
| FR-SHR-03 | **Deposit for shares**: receive funds → liability until allotment → allot (units at price; premium to share premium) or refund |
| FR-SHR-04 | Transfers between holders (approval, stamp duty [A]); a corporate-action log |
| FR-SHR-05 | **Dividends**: declare per class (rate/amount per share, record date) → compute entitlements → WHT (10% [A]) → pay to CASA or bank → unclaimed-dividend tracking |
| FR-SHR-06 | **Preference coupons** accrual and payment |
| FR-SHR-07 | **Reserves transfers**: statutory reserve (% of profit per CBN MFB rules [A]) and regulatory risk reserve (prudential provision minus IFRS impairment, when positive) via approved journals |
| FR-SHR-08 | Reports: register, ownership ≥ 5% (CBN), dividend schedule, capital summary for CBN returns |

## 3. Features & sub-features

Classes · register · deposit for shares · allotment · transfers · dividends/WHT · coupons · reserves · reports.

## 4. Business rules & validations

BR-SHR-01: Σ issued units ≤ authorised. BR-SHR-02: allotment consumes the deposit for shares exactly (any excess is refunded). BR-SHR-03: dividends require board-approval evidence (attachment) plus maker-checker. BR-SHR-04: ownership above CBN thresholds flags for regulatory approval [A].

## 5. Data model

```sql
CREATE TABLE share_class (id BIGSERIAL PRIMARY KEY, name TEXT, type TEXT, nominal NUMERIC(19,6), authorised_units BIGINT, coupon_rate NUMERIC(9,6), gl_capital BIGINT, gl_premium BIGINT);
CREATE TABLE shareholder (id BIGSERIAL PRIMARY KEY, client_id BIGINT, name TEXT, type TEXT, kyc JSONB, status TEXT);
CREATE TABLE share_holding (shareholder_id BIGINT, class_id BIGINT, units BIGINT, PRIMARY KEY (shareholder_id, class_id));
CREATE TABLE share_transaction (id BIGSERIAL PRIMARY KEY, type TEXT, shareholder_id BIGINT, class_id BIGINT, units BIGINT, price NUMERIC(19,6),
  amount NUMERIC(19,6), journal_entry_id BIGINT, command_id BIGINT);
CREATE TABLE dividend (id BIGSERIAL PRIMARY KEY, class_id BIGINT, per_share NUMERIC(19,6), record_date DATE, pay_date DATE, status TEXT, approval_doc TEXT);
CREATE TABLE dividend_entitlement (dividend_id BIGINT, shareholder_id BIGINT, gross NUMERIC(19,6), wht NUMERIC(19,6), net NUMERIC(19,6), paid_at TIMESTAMPTZ,
  PRIMARY KEY (dividend_id, shareholder_id));
```

## 6. API endpoints

`GET/POST /v1/share-classes`, `GET/POST /v1/shareholders`, `POST /v1/shareholders/{id}/share-deposits`, `POST /v1/share-allotments`, `POST /v1/share-transfers`, `POST /v1/dividends`, `POST /v1/dividends/{id}/pay`, `POST /v1/reserve-transfers`, `GET /v1/share-register`.

## 7. Service-layer responsibilities

Register commands, `DividendCalculator`, `ReserveTransferService`.

## 8. Dependencies

clients, accounting, savings-current-accounts, payments-transfers-bills.

## 9. Integration requirements

None external (optional registrar export [A]).

## 10. Authorization & permissions

`MANAGE_SHAREHOLDERS`, `ALLOT_SHARES` (+ `_CHECKER`), `DECLARE_DIVIDEND` (+ `_CHECKER`), `TRANSFER_RESERVES` (+ `_CHECKER`).

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Deposit for shares | Bank/CASA | DEPOSIT_FOR_SHARES (liability) |
| Allotment | DEPOSIT_FOR_SHARES | SHARE_CAPITAL (nominal), SHARE_PREMIUM |
| Dividend declared | RETAINED_EARNINGS | DIVIDEND_PAYABLE, WHT_PAYABLE |
| Dividend paid | DIVIDEND_PAYABLE | Bank/CASA |
| Statutory reserve | RETAINED_EARNINGS | STATUTORY_RESERVE |
| Regulatory risk reserve | RETAINED_EARNINGS | REGULATORY_RISK_RESERVE |

## 12. Error handling & edge cases

Partial allotment; deceased shareholder transmission; unclaimed dividends beyond the statutory period.

## 13. Testing requirements

Register integrity (units vs ledger capital); dividend WHT computation; allotment and premium split.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Current register source (BankOne vs registrar); statutory and regulatory reserve policies.
