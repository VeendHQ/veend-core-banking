# Module: Loan Products

> Phase: P2 · Priority: P0 (parity) · Status: Spec draft
> Fineract reference: `LoanProduct`, `m_product_loan` (+ `_charge`, `_configurable_attributes`, `_variations_borrower_cycle`, `_recalculation_details`), `acc_product_mapping`, `LoanProductsApiResource`
> veend-tenant touchpoints: `createLoanProduct`, `updateLoanProduct`, `getLoanProducts`, `getLoanProductTemplate`, `GET /loans/template` ([V] D2.2 P1–P3, L1). Local `Loanproducts.externalId`.

## 1. Overview & business purpose

A loan product is a versioned template of terms, limits, interest method, repayment strategy, charges and accounting configuration. Loans **freeze** the product terms at approval (`loan.terms`), so later product edits don't alter existing contracts.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-LP-01 | Create and update products with: name, shortName (unique, ≤4), currency, digits, inMultiplesOf; principal min/default/max; numberOfRepayments min/default/max; repaymentEvery + repaymentFrequencyType (DAYS/WEEKS/MONTHS); interestRatePerPeriod min/default/max + interestRateFrequencyType (PER_MONTH/PER_YEAR); interestType (FLAT/DECLINING_BALANCE); amortizationType (EQUAL_INSTALLMENTS/EQUAL_PRINCIPAL); interestCalculationPeriodType (DAILY/SAME_AS_REPAYMENT_PERIOD); daysInYearType (ACTUAL/360/364/365); daysInMonthType (ACTUAL/30); transactionProcessingStrategyCode; graces; charges[]; accountingRule; GL mappings by role; canUseForTopup; holiday policy; rounding mode |
| FR-LP-02 | **Product template** endpoint: defaults and options (the Fineract `/loanproducts/template` equivalent) |
| FR-LP-03 | **Loan template for a client** (`/loans/template?clientId&productId`): resolved defaults including product charges. Used by calculate and create (compat L1) |
| FR-LP-04 | Version products. Each update creates `loan_product_version`; loans reference the version used at approval |
| FR-LP-05 | Activate and deactivate a product (start and close dates) |
| FR-LP-06 | Compat P1–P3 shapes and field names (`interestRatePerPeriod`, `numberOfRepayments`, `minPrincipal`, …). This includes accepting Fineract enum ids (e.g. `interestType:1`, `accountingRule:4`) and the 12 GL-id parameters (`fundSourceAccountId`, …) |
| FR-LP-07 | A product created via compat with no explicit GL ids gets mappings from the **tenant's default product accounting template** instead of hard-coded ids ([V] DS-08) |
| FR-LP-08 | **Quick loan product creation** (recovery-team F-21): a wizard/API that creates a product from a template (e.g. "payroll loan", "micro loan", "SME loan", "payday") with a few inputs (rate, tenor range, amount range, fees). All other terms and GL mappings come from the template. The product is still versioned and validated (BR-LP-01..06) |
| FR-LP-09 | **Mortgage loan products** (F-22): declining-balance EMI, long tenor (up to 30 years), optional interest-only or construction-draw period (multi-tranche, P9), mandatory property collateral with an LTV limit and insurance ([collateral-guarantors](collateral-guarantors.md) FR-COL-08), equity-contribution check, optional variable rate index [A] |
| FR-LP-10 | Product classification attributes for regulatory reporting: CBN loan type (micro/SME/staff/other), lending model, default economic sector (overridable per loan) |

## 3. Features

| Feature | Sub-feature | MVP? | Notes |
|---|---|---|---|
| Terms | Principal, tenure, frequency, rate ranges | Yes | |
| Interest | FLAT, DECLINING_BALANCE (EMI and equal principal) | Yes | Production uses FLAT ([V] `interestType: 1`); declining needed for future products |
| Interest recalculation | Daily recalculation on early or late payment | No | P9; [A] verify it is disabled in production (`isInterestRecalculationEnabled`) |
| Strategy | `mifos-standard-strategy` + production strategies | Yes | [A] read `loan_transaction_strategy_code` for every live product. veend-tenant takes `transactionProcessingStrategyOptions[0]` ([V] `lendlot/index.js:868`), so the actual value depends on Fineract's option ordering |
| Charges | Attach charge definitions (see [charges](charges.md)) | Yes | |
| Accounting | NONE, CASH, ACCRUAL_UPFRONT, ACCRUAL_PERIODIC; role mappings; payment-type overrides | Yes | |
| Borrower-cycle variations | | No | |
| Multi-tranche | | No | `editDisbursements` usage is broken today ([V] DF-08) |
| Down payment, buy-down, capitalised income | | No | |
| Hidden/system products (on-lending) | `is_system` flag hides a product from customer listings | Yes | veend-tenant creates a hidden "onlending" product ([V] F5) |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-LP-01 | `min ≤ default ≤ max` for principal, repayments and rate |
| BR-LP-02 | `shortName` is unique per tenant. veend-tenant derives it from the first 3 chars of the name ([V] `:867`), which can collide. **The compat layer auto-suffixes on collision** (`ABC`, `AB1`, …) and returns the actual value |
| BR-LP-03 | When `accountingRule ≠ NONE`, every **required role** for that rule must be mapped to an active detail GL account of the correct type. Required roles: FUND_SOURCE(asset), LOAN_PORTFOLIO(asset), INTEREST_ON_LOANS(income), INCOME_FROM_FEES(income), INCOME_FROM_PENALTIES(income), LOSSES_WRITTEN_OFF(expense), OVERPAYMENT(liability), INCOME_FROM_RECOVERY(income), TRANSFERS_SUSPENSE(asset). For accrual rules, also INTEREST/FEES/PENALTIES_RECEIVABLE(asset) |
| BR-LP-04 | Changing the accounting rule is forbidden while non-closed loans exist on the product |
| BR-LP-05 | Product edits never alter existing loans. Loans keep their frozen `terms` |
| BR-LP-06 | The currency must be enabled. Digits come from the currency unless overridden down |

## 5. Data model

```sql
CREATE TABLE loan_product (
  id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL, short_name VARCHAR(4) UNIQUE NOT NULL, external_id TEXT UNIQUE,
  legacy_fineract_id BIGINT UNIQUE, is_system BOOLEAN DEFAULT false, status TEXT NOT NULL DEFAULT 'ACTIVE',
  start_date DATE, close_date DATE, current_version INT NOT NULL);
CREATE TABLE loan_product_version (
  product_id BIGINT REFERENCES loan_product(id), version INT, created_at TIMESTAMPTZ DEFAULT now(), created_by TEXT,
  terms JSONB NOT NULL,          -- validated by a zod schema: LoanProductTerms (all FR-LP-01 fields)
  PRIMARY KEY (product_id, version));
CREATE TABLE loan_product_charge (product_id BIGINT REFERENCES loan_product(id), charge_id BIGINT REFERENCES charge(id), PRIMARY KEY (product_id, charge_id));
CREATE TABLE product_gl_mapping (
  product_id BIGINT REFERENCES loan_product(id), role TEXT NOT NULL, gl_account_id BIGINT NOT NULL REFERENCES gl_account(id),
  payment_type_id BIGINT REFERENCES payment_type(id),   -- NULL = default for role
  charge_id BIGINT REFERENCES charge(id),               -- optional fee/penalty income override per charge
  PRIMARY KEY (product_id, role, COALESCE(payment_type_id,0), COALESCE(charge_id,0)));
```

`LoanProductTerms` (TypeScript, abbreviated):

```ts
interface LoanProductTerms {
  currency: { code: string; digits: number; inMultiplesOf: number };
  principal: { min: Decimal; default: Decimal; max: Decimal };
  repayments: { min: number; default: number; max: number; every: number; frequency: 'DAYS'|'WEEKS'|'MONTHS' };
  interest: { min: Decimal; default: Decimal; max: Decimal; per: 'MONTH'|'YEAR';
              method: 'FLAT'|'DECLINING_BALANCE'; amortization: 'EQUAL_INSTALLMENTS'|'EQUAL_PRINCIPAL';
              calculationPeriod: 'DAILY'|'SAME_AS_REPAYMENT_PERIOD';
              daysInYear: 'ACTUAL'|360|364|365; daysInMonth: 'ACTUAL'|30;
              allowPartialPeriodInterest: boolean };
  grace: { principal: number; interestPayment: number; interestCharged: number; arrearsTolerance: Decimal; daysInArrearsGrace: number };
  strategy: string;                       // e.g. 'mifos-standard-strategy'
  rounding: 'HALF_EVEN'|'HALF_UP'|'HALF_DOWN';
  installmentInMultiplesOf?: Decimal;
  accountingRule: 'NONE'|'CASH'|'ACCRUAL_UPFRONT'|'ACCRUAL_PERIODIC';
  holidayPolicy: 'SAME_DAY'|'NEXT_WORKING_DAY'|'PREVIOUS_WORKING_DAY'|'NEXT_REPAYMENT_DAY';
  canUseForTopup: boolean;
  overdue: { penaltyChargeIds: number[]; npaAfterDays?: number };
}
```

## 6. API endpoints

- **Native:**
  - `GET /v1/loan-products?includeSystem=false`, `POST /v1/loan-products`
  - `GET/PATCH /v1/loan-products/{id}`, `GET /v1/loan-products/{id}/versions/{v}`
  - `GET /v1/loan-products/template`, `GET /v1/loan-products/{id}/template?clientId=`
  - `GET/PUT /v1/loan-products/{id}/gl-mappings`
- **Compat:**
  - `GET /loanproducts`, `GET /loanproducts/{id}?template=true`, `GET /loanproducts/template`, `POST /loanproducts` (returns `{resourceId}`), `PUT /loanproducts/{id}` (returns `{resourceId, changes}`)
  - `GET /loans/template?...&templateType=individual`: response includes `termPeriodFrequencyType{id}`, `repaymentEvery`, `repaymentFrequencyType{id}`, `interestRatePerPeriod`, `amortizationType{id}`, `interestType{id}`, `interestCalculationPeriodType{id}`, `transactionProcessingStrategyCode` and `transactionProcessingStrategyId` [A: whichever field the production Fineract version returns], `charges[{chargeId, amount, …}]`, `transactionProcessingStrategyOptions[]`

## 7. Service-layer responsibilities

Commands: `CreateLoanProduct`, `UpdateLoanProduct` (new version), `SetProductGlMappings`, `ActivateProduct`, `DeactivateProduct`. `ProductTemplateService` builds the template, with Fineract enum id ↔ code mapping tables for compat.

## 8. Dependencies

organisation (currency, payment types), charges, accounting (GL accounts). Consumed by the schedule engine and loan-lifecycle.

## 9. Integration

Event `loan_product.updated`. veend-tenant's `syncLoanProductFrom` job could switch to this event (M4).

## 10. Authorization

`CREATE/UPDATE/READ_LOANPRODUCT`, `UPDATE_PRODUCTMAPPING` (maker-checker recommended because it changes GL routing).

## 11. Transactional & accounting implications

The mappings determine every posting for loans on the product. Mapping changes apply to postings **after** the change, and are audited.

## 12. Edge cases

- Compat `updateLoanProduct` echoes the whole template ([V] `:921-968`), including fields we don't support. Unsupported values that equal the defaults are ignored; non-default values return 400 with the field named.
- veend-tenant reads `currency.code` from the template ([V] `:927`), so it must be present.

## 13. Testing

- Validation unit tests for BR-LP-01..06.
- Compat contract tests with recorded payloads.
- **Migration test:** every live LendLot product converts to `LoanProductTerms` and reproduces Fineract's `calculateLoanSchedule` output for its default terms (golden).

## 14. Status & gaps

- Not implemented.
- [A] Export all production `m_product_loan` rows (strategy, rounding, days-in-year/month, recalculation, grace, charges) to confirm the MVP subset is sufficient.
