# Module: Reporting

> Phase: P5 · Priority: P1 (P0 for the two compat reports) · Status: Spec draft
> Fineract reference: `RunreportsApiResource`, `stretchy_report`, `stretchy_parameter`; stock reports "Active Loans - Summary" ([V] `0002_initial_data.xml:12565`), "Aging Detail" (`:12684`)
> veend-tenant touchpoints: `GET /runreports/ActiveLoansSummary` (custom report, **not stock**) and `GET /runreports/Aging%20Detail?R_officeId=1`, both read positionally ([V] D2.2 §3). Consumers: `Dashboard.portfolio`, `getAgingReport`, `Loans.getLoanReports`, `jobs.updateMetrics`, `arrearsNotification`, `checkOverdueTenureLoans`

## 1. Overview & business purpose

This module provides versioned, tested financial and portfolio reports for finance, operations, auditors and regulators, plus exact reproductions of the two LendLot reports veend-tenant depends on.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-RPT-01 | Report registry: code, name, category, parameters (typed), output columns (ordered), permission and SQL module, all **version-controlled in code**. No runtime SQL authoring |
| FR-RPT-02 | Output in JSON (native `{columns, rows}`), CSV, and Fineract `genericResultSet` shape (`{columnHeaders:[{columnName, columnType}], data:[{row:[...]}]}`) for compat |
| FR-RPT-03 | Compat `ActiveLoansSummary`: reproduce production column order and semantics exactly (the SQL must be extracted from production; D2.2 §2.5) |
| FR-RPT-04 | Compat `Aging Detail`: 15 columns in the order listed in D2.2 §3, account number zero-padded |
| FR-RPT-05 | Native MVP catalogue (see §3) |
| FR-RPT-06 | Heavy reports run on the read replica, or from COB snapshot tables for "as of COB date" |
| FR-RPT-07 | Scheduled report delivery (email or SFTP): P9 |
| FR-RPT-08 | **Report metadata** on every output (RT-06): as-of date, generated-at, data refresh time (COB date or live), definition version, filters and exclusions, tenant. Re-running a report for a past as-of date reproduces the same figures, except for documented back-dated adjustments, which are listed |
| FR-RPT-09 | **Statements** (RT-05): loan and deposit account statements with opening balance, booking date, value date, reference, description, debit, credit, running balance and closing balance, for a period. Tested: closing balance = ledger balance at the same cut-off |
| FR-RPT-10 | **Call-over report** (RT-07): per business date, every posting with GL account, debit/credit, reference, entry code, maker, checker, approval time, terminal/IP and narration. Plus a sign-off record for the reviewer |
| FR-RPT-11 | **CBN MFB monthly returns** (RT-08), generated from GL + classification + maturity buckets, with a validation step (totals tie to TB) before export in the CBN template layout |
| FR-RPT-13 | **Multiple and batch statements** (recovery-team F-39): generate statements for many accounts in one job (by branch, product or list); a batch-statement queue with status, view/download per batch, and optional email delivery. Statement layout versions are configurable (the guide mentions "Statement versions 1.0 and 2.0"). Every version must reconcile to the ledger |
| FR-RPT-14 | **Management reports** (F-43): collections by channel and officer, maturity pipeline (loans and deposits), income analysis, recovery performance, deposit mix and cost of funds |
| FR-RPT-12 | **Read-only reporting views** (RT-11) in schema `rpt_views` for Metabase/BI, each with a definition entry in the catalogue |

## 3. Report catalogue (MVP)

| Code | Category | Parameters | Source |
|---|---|---|---|
| `compat.ActiveLoansSummary` | Compat | R_currencyId, R_officeId, R_parType, … | `rpt_active_loans_summary` snapshot + live |
| `compat.AgingDetail` | Compat | R_officeId | `rpt_aging_detail` snapshot (COB) |
| `gl.trial_balance` | Finance | asOf, office | `gl_balance` / lines |
| `gl.general_ledger` | Finance | account, from, to | `journal_line` |
| `gl.balance_sheet`, `gl.income_statement` | Finance | asOf / from-to | lines by account type/category |
| `gl.cash_flow` | Finance | from, to | GL movements by category (FR-ACC-11) |
| `portfolio.summary` | Portfolio | asOf, product, office | loans |
| `portfolio.par` | Portfolio | asOf, buckets | `loan_delinquency` |
| `portfolio.aging` | Portfolio | asOf | `loan_delinquency` |
| `portfolio.disbursements` | Ops | from, to, product | transactions type 1 |
| `portfolio.collections` | Ops | from, to, paymentType | transactions type 2/8 |
| `portfolio.writeoffs`, `portfolio.recoveries` | Finance | from, to | |
| `portfolio.provisioning` | Finance | runDate | provisioning tables |
| `recon.clearing_account` | Finance | paymentType, from, to | lines on the clearing GL with `external_ref` |
| `audit.commands` | Audit | from, to, actor, type | `command_log` |
| `audit.callover` | Audit | businessDate, office | journal lines + command_log |
| `statement.loan`, `statement.deposit` | Customer | accountId, from, to | transactions + running balance |
| `portfolio.classification` | Risk | asOf, basis | `loan_delinquency` + provisioning basis (BR-WO-04a) |
| `cbn.mmfbr300` (assets & liabilities), `cbn.mmfbr1000` (P&L), `cbn.mmfbr761` (classification), `cbn.mmfbr771` (NPL schedule), `cbn.mmfbr763` (loan maturity profile) | Regulatory | reportingDate | GL + classification; ties to TB |
| `cbn.mmfbr201/202/141` (deposit structure, insured deposits, other deposits) | Regulatory | reportingDate | Only when deposit-taking is in scope ([input review §8](../09-input-reviews/recovery-team-2026-10.md#8-open-decisions-raised-by-this-submission)) |
| Remaining MMFBR schedules (001, 221, 311, 312, 321, 322, 451, 501, 641, 642, 651, 711, 746, 762, 764, 811, 933, 951, 980, 996, AML/KYC) and credit-bureau files | Regulatory | reportingDate | **In scope (ADR-010)**: delivered by [regulatory-reporting](regulatory-reporting.md) in P7-G. Some need non-CBA inputs (placements, borrowings, staff counts), captured via the treasury/manual-input screens there |

## 4. Business rules

- **BR-RPT-01.** Monetary columns are rendered at currency digits. Compat reports render numbers as strings exactly as Fineract does (veend-tenant applies `Number(...)`, `parseFloat`, `parseInt`; [V] D2.2 §3).
- **BR-RPT-02.** "As of" reports use the COB snapshot when `asOf ≤ cobDate`; otherwise live data with a disclaimer column.
- **BR-RPT-03.** Office filter: compat `R_officeId=1` means the head office **and descendants**, which is Fineract hierarchy semantics.

## 5. Data model

```sql
CREATE TABLE rpt_aging_detail (cob_date DATE, loan_id BIGINT, office_name TEXT, currency TEXT, client_account_no TEXT, client_name TEXT,
  loan_account_no TEXT, loan_amount NUMERIC(19,6), original_principal NUMERIC(19,6), original_interest NUMERIC(19,6), principal_paid NUMERIC(19,6),
  interest_paid NUMERIC(19,6), principal_overdue NUMERIC(19,6), interest_overdue NUMERIC(19,6), days_in_arrears INT, weeks_band TEXT, days_band TEXT,
  PRIMARY KEY (cob_date, loan_id));
CREATE TABLE rpt_active_loans_summary (cob_date DATE, office_id BIGINT, currency TEXT, cols JSONB, PRIMARY KEY (cob_date, office_id, currency));
```

## 6. API endpoints

- Native: `GET /v1/reports`, `GET /v1/reports/{code}?…` (`Accept: application/json | text/csv`).
- Compat: `GET /runreports/{name}?R_*=…` (`genericResultSet=true` default).

## 7. Service-layer responsibilities

`ReportRegistry`, `ReportRunner` (parameter validation, read-replica routing, output formatter), `CompatReportAdapter`, `SnapshotWriter` (COB).

## 8. Dependencies

accounting, loan-lifecycle, loan-transactions, cob-jobs (snapshots), platform-core.

## 9. Integration

ERP export (GL daily extract): P5 [A target system]. Regulator returns (CBN / NDIC formats): future [A].

## 10. Authorization

`RUN_REPORT_<code>` per report. Category grants: `RUN_REPORT_FINANCE_*`, etc.

## 11. Transactional & accounting implications

Read-only.

## 12. Edge cases

- Positional compat columns must never change order. Contract tests pin `columnHeaders` names and order.
- Bands text must match exactly (e.g. `"0 - 30"`), because veend-tenant string-compares it ([V] `Dashboard.js:5472`).

## 13. Testing

- Compat: run the same seeded portfolio through LendLot (golden) and Fineract-Mod, and diff the full report output.
- Finance reports: reconcile the trial balance against the account ledgers.

## 14. Status & gaps

- Not implemented.
- [A] **Blocking for compat:** the production SQL of `ActiveLoansSummary` and the production column list for `Aging Detail` (the stock definition may have been customised).
