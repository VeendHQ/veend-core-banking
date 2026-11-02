# Module: Fixed Assets and Amortization

> Phase: **P7-E** (enterprise track) · Priority: P1 · Status: Spec draft
> Scope origin: F-05, F-06 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.1 "Accounting Management menu includes GL Accounts, Account Categories, Fixed Assets and Amortization"; Pettysave BS fixed assets/accumulated depreciation, prepayments, goodwill/intangibles [V]; TB asset classes and accumulated-depreciation GLs [V]
> Fineract reference: none

## 1. Overview & business purpose

This module keeps the fixed-asset register and depreciation, plus amortization schedules for prepayments, intangibles (goodwill, software), deferred income (e.g. loan fees amortised over the term, when that policy is chosen) and preliminary expenses. All are posted to the GL automatically by period-end jobs.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-FA-01 | Asset classes (Furniture & Fittings, Motor Vehicles, Office Equipment, Computer Equipment, Plant & Machinery, Leasehold Improvement, Software, …) with method (straight-line or reducing balance), useful life, residual %, and GL mapping (cost, accumulated depreciation, depreciation expense, disposal gain/loss) |
| FR-FA-02 | Asset register: tag, description, class, branch, custodian, acquisition date and cost, supplier/invoice, status |
| FR-FA-03 | Acquisition posting (from payable, bank or capitalised), additions, revaluation (optional), transfer between branches |
| FR-FA-04 | Monthly depreciation run (idempotent per period) with a preview, approval and posting |
| FR-FA-05 | Disposal/write-off: proceeds, NBV, gain or loss posting |
| FR-FA-06 | **Amortization schedules** for prepayments (rent, insurance), intangibles and deferred items: amount, start, periods, GL pair; monthly run |
| FR-FA-07 | Reports: asset register, depreciation schedule, NBV by class (feeds the CBN 300 fixed-asset lines), amortization schedule |

## 3. Features & sub-features

Asset classes · register · acquisition/addition/transfer · depreciation run · disposal · prepaid/intangible amortization · reports.

## 4. Business rules & validations

BR-FA-01: depreciation starts in the acquisition month (or the next month, per policy [A]). BR-FA-02: no depreciation below residual value. BR-FA-03: one posted run per period; a re-run replaces the draft only. BR-FA-04: posting goes into open periods only.

## 5. Data model

```sql
CREATE TABLE asset_class (id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE, method TEXT, life_months INT, residual_pct NUMERIC(9,6),
  cost_gl BIGINT, accum_dep_gl BIGINT, dep_expense_gl BIGINT, disposal_gl BIGINT);
CREATE TABLE fixed_asset (id BIGSERIAL PRIMARY KEY, tag TEXT UNIQUE, class_id BIGINT, office_id BIGINT, description TEXT, custodian TEXT,
  acquired_on DATE, cost NUMERIC(19,6), accumulated_dep NUMERIC(19,6) DEFAULT 0, status TEXT, disposed_on DATE, proceeds NUMERIC(19,6));
CREATE TABLE amortization_schedule (id BIGSERIAL PRIMARY KEY, type TEXT, description TEXT, total NUMERIC(19,6), start_period DATE, periods INT,
  debit_gl BIGINT, credit_gl BIGINT, amortised NUMERIC(19,6) DEFAULT 0, status TEXT);
CREATE TABLE period_run (id BIGSERIAL PRIMARY KEY, kind TEXT, period DATE, status TEXT, journal_entry_id BIGINT, UNIQUE (kind, period));
```

## 6. API endpoints

`GET/POST /v1/asset-classes`, `GET/POST /v1/fixed-assets`, `POST /v1/fixed-assets/{id}/disposal`, `POST /v1/fixed-assets/{id}/transfer`, `POST /v1/depreciation-runs` (`{period}`), `POST /v1/depreciation-runs/{id}/post`, `GET/POST /v1/amortization-schedules`, `POST /v1/amortization-runs`.

## 7. Service-layer responsibilities

`DepreciationCalculator` (pure), `AmortizationCalculator`, period-run commands, register queries.

## 8. Dependencies

accounting (GL, closures), organisation (branches), platform-core.

## 9. Integration requirements

Optional import from the procurement/AP system [A].

## 10. Authorization & permissions

`MANAGE_FIXEDASSETS`, `RUN_DEPRECIATION` (+ `_CHECKER`), `DISPOSE_ASSET` (+ `_CHECKER`), `MANAGE_AMORTIZATION`.

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Acquisition | Asset cost GL | Accounts payable / bank |
| Depreciation | Depreciation expense | Accumulated depreciation |
| Disposal | Accumulated depreciation, bank (proceeds), loss | Asset cost, gain |
| Prepayment amortization | Expense (rent, insurance) | Prepayment asset |

## 12. Error handling & edge cases

Mid-month acquisition and pro-rating per policy; a depreciation run in a closed period → 423; a disposed asset is excluded from later runs.

## 13. Testing requirements

Calculator unit tests (SL/RB, residual); run idempotency; TB impact equals the run report.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Accounting policy (methods, lives), opening register from BankOne/finance spreadsheets.
