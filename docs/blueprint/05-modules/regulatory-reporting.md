# Module: Regulatory and Credit-Bureau Reporting

> Phase: **P7-G** (core CBN returns already in P5 via reporting) · Priority: P0 for BANK tenants · Status: Spec draft
> Scope origin: F-44, F-45 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.5 "CBN Reports", "Credit Bureau Reports", §12 ("How are CBN and credit-bureau reporting definitions validated?"); Pettysave CBN workbook: MMFBR 300, 1000, 001, 141, 201, 202, 221, 311, 312, 321, 322, 451, 501, 641, 642, 651, 711, 746, 761, 762, 763, 764, 771, 811, 933, 951, 980, 996, AML/KYC [V sheet list]
> Fineract reference: `CreditBureau*` ([V] `fineract-provider/.../infrastructure/creditbureau/domain/CreditBureau.java`), stretchy reports

## 1. Overview & business purpose

This module produces regulator and bureau submissions from validated CBA data, with traceability to the GL and to loans:
- the **full CBN MFB monthly return set (MMFBR)**;
- NDIC returns (insured deposits, premium basis);
- AML/KYC returns;
- **credit-bureau** data submissions to CRC, FirstCentral and CreditRegistry (borrower, facility and repayment data), plus bureau enquiries for appraisal.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-REG-01 | **Return definitions as versioned code**: each MMFBR schedule maps its line codes (e.g. 10710 Micro Loans, 10770 Specific Provision, 20130 Time Deposits) to GL account sets, classification buckets or maturity buckets. Mapping tables are maintained by finance and compliance with an audit trail |
| FR-REG-02 | Generation for a reporting date (month-end COB snapshot), in the CBN template layout (XLSX/XML as required [A: current CBN eFASS/portal format]) |
| FR-REG-03 | **Validation rules before export**: (a) MMFBR 300 assets = liabilities + capital; (b) 300 loan lines = 761 total = Σ loan sub-ledger; (c) 761 buckets = classification engine output on the **printed exposure basis** (BR-WO-04a); (d) provisions in 300 = provisioning entry; (e) 1000 P&L ties to the GL income/expense for the period, with cumulative and monthly clearly labelled (input review I-2); (f) loan counts consistent across 711, 763 and 771 (input review I-1). A failing rule blocks submission unless overridden by compliance with a reason |
| FR-REG-04 | Submission log: generated file hash, approver, submitted at, portal reference, resubmissions |
| FR-REG-05 | **NDIC**: insured-deposit schedule (MMFBR 202 analogue), premium computation support |
| FR-REG-06 | **AML/KYC**: customer risk distribution, CTR (cash transactions ≥ threshold) and STR workflow hand-off to NFIU [A: goAML]; the AML/KYC return sheet |
| FR-REG-07 | **Credit bureau submissions**: monthly borrower and facility files in each bureau's format (individual and corporate), including classification, DPD, balances, guarantors, collateral; error-report ingestion and resubmission |
| FR-REG-08 | **Bureau enquiries** at appraisal: consent-gated request → report stored (encrypted) → summary to appraisal (FR-LN-14). veend-tenant's existing creditcheck integration can be reused through this API [A] |
| FR-REG-09 | Reproducibility: any past return can be regenerated from snapshots and match the submitted hash (or show documented adjustments) |

## 3. Features & sub-features

Return catalogue (all MMFBR schedules + AML/KYC) · mapping management · generation · validation · approval and submission log · NDIC · AML (CTR/STR) · bureau submissions · bureau enquiries · reproducibility.

## 4. Business rules & validations

BR-REG-01: returns are generated only from a **closed** month (GL closure posted). BR-REG-02: amounts in N'000 with documented rounding (half-up to the nearest thousand), plus a rounding-difference line reconciled. BR-REG-03: the classification basis and provisioning rates are printed on 761/771. BR-REG-04: bureau files exclude customers without valid consent where the law requires it [A].

## 5. Data model

```sql
CREATE TABLE return_definition (code TEXT, version INT, name TEXT, layout JSONB, PRIMARY KEY (code, version));
CREATE TABLE return_line_mapping (return_code TEXT, version INT, line_code TEXT, source TEXT, selector JSONB, PRIMARY KEY (return_code, version, line_code));
CREATE TABLE return_run (id BIGSERIAL PRIMARY KEY, return_code TEXT, version INT, reporting_date DATE, status TEXT, validation JSONB,
  file_hash CHAR(64), approved_by TEXT, submitted_at TIMESTAMPTZ, portal_ref TEXT);
CREATE TABLE bureau_submission (id BIGSERIAL PRIMARY KEY, bureau TEXT, period DATE, file_hash CHAR(64), records INT, status TEXT, error_report JSONB);
CREATE TABLE bureau_enquiry (id BIGSERIAL PRIMARY KEY, client_id BIGINT, bureau TEXT, consent_ref TEXT, report_enc BYTEA, summary JSONB, at TIMESTAMPTZ);
```

## 6. API endpoints

`GET /v1/regulatory-returns` (catalogue), `POST /v1/regulatory-returns/{code}/runs` (`{reportingDate}`), `GET …/runs/{id}` (validation and file), `POST …/runs/{id}/approve`, `POST …/runs/{id}/submitted` (portal ref), `GET/PUT /v1/regulatory-returns/{code}/mappings`, `POST /v1/bureau-submissions`, `POST /v1/bureau-enquiries`.

## 7. Service-layer responsibilities

`ReturnGenerator` (mapping-driven), `ReturnValidator` (rules a–f), `BureauFileBuilder` per bureau, `BureauEnquiryClient`, CTR detector (COB).

## 8. Dependencies

accounting, reporting (snapshots), cob-jobs (classification), writeoff-provisioning, savings-current-accounts, fixed-deposits, clients, collateral-guarantors.

## 9. Integration requirements

CBN reporting portal [A format], NDIC portal, NFIU goAML [A], bureau SFTP/APIs.

## 10. Authorization & permissions

`GENERATE_RETURN`, `APPROVE_RETURN` (`_CHECKER`, compliance), `MANAGE_RETURNMAPPINGS` (+ `_CHECKER`), `SUBMIT_BUREAU`, `ENQUIRE_BUREAU` (consent required).

## 11. Transactional & accounting implications

None (read-only), except that validations depend on a closed GL period.

## 12. Error handling & edge cases

A late-posted adjustment after submission → a resubmission run with a diff; bureau rejections → corrected records queued; a mapping change mid-year → versioned, and old runs keep their version.

## 13. Testing requirements

Rebuild Pettysave's 30-Sep-2026 returns from migrated data and compare line by line, explaining the known 761 discrepancy (input review §2.2). Validation-rule unit tests. Bureau file schema validation.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Current CBN submission format and portal; bureau contracts; NFIU integration; compliance owner for mappings.
