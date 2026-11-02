# Module: Collateral and Guarantors (incl. mortgage support)

> Phase: **P7-R** (credit-risk track) · Priority: P1 · Status: Spec draft
> Scope origin: F-22, F-27, F-62 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.2 "collateral requirements", §8 "collateral"; PAR columns Collateral, Collateral Value, Guarantor 1/2 (name, phone, address, % and value of exposure) [V structure]
> Fineract reference: `ClientCollateralManagement`, `LoanCollateralManagement` ([V] `fineract-loan/.../collateralmanagement`), `Guarantor` ([V] `fineract-provider/.../loanaccount/guarantor/domain/Guarantor.java`)

## 1. Overview & business purpose

This module records and values security for loans: physical collateral (property for mortgages, vehicles, equipment), financial collateral (FD/CASA liens, cash-backed), payroll/IPPIS assignment, and personal guarantors with exposure shares. It feeds appraisal, provisioning (collateral haircuts, where policy allows), recovery and reporting.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-COL-01 | Collateral types (configurable): REAL_ESTATE, VEHICLE, EQUIPMENT, FD_LIEN, CASA_LIEN, SALARY_DOMICILIATION/IPPIS, SHARES, OTHER. Each has a haircut %, revaluation frequency and required documents |
| FR-COL-02 | Collateral register per client: description, documents (title, registration), valuation (value, date, valuer), location, insurance (policy, expiry), perfection status (registered charge, NCR/collateral registry ref [A]) |
| FR-COL-03 | Pledge collateral to loans (many-to-many, with a pledged value). Coverage ratio = Σ (value × (1 − haircut)) / exposure |
| FR-COL-04 | **Financial collateral** places a hold on the FD/CASA (via the deposit modules), released at loan closure |
| FR-COL-05 | **Guarantors**: an existing client or an external person; relationship; guaranteed % or amount; optional guarantor deposit hold; contact details; exposure value computed |
| FR-COL-06 | Product rules: minimum coverage, required guarantor count, LTV for mortgages; enforced at approval |
| FR-COL-07 | Revaluation reminders; insurance expiry alerts; release of collateral at closure (with documentation checklist) |
| FR-COL-08 | **Mortgage support**: property collateral with LTV, insurance, title perfection, long-tenor declining-balance product ([loan-products](loan-products.md) FR-LP-09) |
| FR-COL-09 | Recovery: realisation of collateral or calling a guarantee → recovery transaction (proceeds), linked to [recovery-collections](recovery-collections.md) |

## 3. Features & sub-features

Types · register · valuations · pledges · coverage · holds · guarantors · product enforcement · alerts · release · realisation · mortgage specifics.

## 4. Business rules & validations

BR-COL-01: a loan approval fails if product coverage, LTV or guarantor rules are unmet (maker-checker override with reason). BR-COL-02: collateral pledged across loans may not exceed its value. BR-COL-03: guarantor exposure % across guarantors ≤ 100%. BR-COL-04: release only on a closed loan or with approval.

## 5. Data model

```sql
CREATE TABLE collateral_type (code TEXT PRIMARY KEY, name TEXT, haircut_pct NUMERIC(9,6), revalue_months INT, docs JSONB);
CREATE TABLE collateral (id BIGSERIAL PRIMARY KEY, client_id BIGINT, type_code TEXT, description TEXT, value NUMERIC(19,6), valued_on DATE, valuer TEXT,
  insurance JSONB, perfection JSONB, hold_ref TEXT, status TEXT, documents JSONB);
CREATE TABLE loan_collateral (loan_id BIGINT, collateral_id BIGINT, pledged_value NUMERIC(19,6), PRIMARY KEY (loan_id, collateral_id));
CREATE TABLE guarantor (id BIGSERIAL PRIMARY KEY, loan_id BIGINT, client_id BIGINT, name TEXT, phone_enc BYTEA, address_enc BYTEA, relationship TEXT,
  guaranteed_pct NUMERIC(9,6), guaranteed_amount NUMERIC(19,6), hold_ref TEXT, status TEXT);
```

## 6. API endpoints

`GET/POST /v1/collateral-types`, `GET/POST /v1/clients/{id}/collateral`, `POST /v1/collateral/{id}/valuations`, `POST /v1/loans/{id}/collateral`, `DELETE /v1/loans/{id}/collateral/{cid}`, `GET/POST /v1/loans/{id}/guarantors`, `GET /v1/loans/{id}/coverage`.

## 7. Service-layer responsibilities

`CoverageCalculator`, `CollateralPolicyGuard` (hook on `ApproveLoan`), hold integration, alert jobs (COB).

## 8. Dependencies

clients, loan-lifecycle, savings-current-accounts, fixed-deposits, writeoff-provisioning, recovery-collections.

## 9. Integration requirements

National Collateral Registry (movables) [A]; property title verification (manual).

## 10. Authorization & permissions

`MANAGE_COLLATERAL`, `VALUE_COLLATERAL`, `RELEASE_COLLATERAL` (+ `_CHECKER`), `MANAGE_GUARANTORS`, `OVERRIDE_COLLATERALPOLICY` (`_CHECKER`).

## 11. Transactional & accounting implications

No GL impact on pledge, apart from memo/off-balance-sheet tracking (CBN MMFBR 996 if applicable). Realisation proceeds post as a recovery or repayment. Holds affect available balances only.

## 12. Error handling & edge cases

Collateral shared by co-borrowers; an expired valuation at approval (blocked); a guarantor who is also a borrower in arrears (warning).

## 13. Testing requirements

Coverage calculations; approval-guard tests; hold/release lifecycle.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Pettysave collateral and guarantor data quality (803 PAR rows carry collateral text); valuation policy.
