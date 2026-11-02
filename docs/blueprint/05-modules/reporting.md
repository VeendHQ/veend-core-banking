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
| FR-RPT-07 | Scheduled report delivery (email or SFTP): P7 |

## 3. Report catalogue (MVP)

| Code | Category | Parameters | Source |
|---|---|---|---|
| `compat.ActiveLoansSummary` | Compat | R_currencyId, R_officeId, R_parType, … | `rpt_active_loans_summary` snapshot + live |
| `compat.AgingDetail` | Compat | R_officeId | `rpt_aging_detail` snapshot (COB) |
| `gl.trial_balance` | Finance | asOf, office | `gl_balance` / lines |
| `gl.general_ledger` | Finance | account, from, to | `journal_line` |
| `gl.balance_sheet`, `gl.income_statement` | Finance | asOf / from-to | lines by account type |
| `portfolio.summary` | Portfolio | asOf, product, office | loans |
| `portfolio.par` | Portfolio | asOf, buckets | `loan_delinquency` |
| `portfolio.aging` | Portfolio | asOf | `loan_delinquency` |
| `portfolio.disbursements` | Ops | from, to, product | transactions type 1 |
| `portfolio.collections` | Ops | from, to, paymentType | transactions type 2/8 |
| `portfolio.writeoffs`, `portfolio.recoveries` | Finance | from, to | |
| `portfolio.provisioning` | Finance | runDate | provisioning tables |
| `recon.clearing_account` | Finance | paymentType, from, to | lines on the clearing GL with `external_ref` |
| `audit.commands` | Audit | from, to, actor, type | `command_log` |

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
