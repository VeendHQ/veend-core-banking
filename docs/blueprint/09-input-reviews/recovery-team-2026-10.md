# Input Review: Recovery Team Submission (October 2026)

> Source folder: `docs/cba-notes-recovery-team/` (received 2026-11-02). Reviewed 2026-11-02.
> **Data-handling note:** the PAR report contains customer names, BVNs, phone numbers, addresses and guarantor details. This review uses **aggregates only**. The source files should not be committed to git (see §7).

## 1. What was received

| File | What it is | Entity / as-of | Notes |
|---|---|---|---|
| `VeendHQ_Core_Banking_Application_Blueprint.docx` / `.pptx` / `.pdf` | A vendor-neutral implementation guide: architecture layers, module usage, posting examples, payment integration, reporting, recovery data model, reconciliation controls, roadmap, vendor-readiness questions, UAT | VeendHQ | Written against the **menus of a vendor CBS** (Accounting/Posting Mgmt, Product Mgmt incl. fixed deposits, commitment savings, mortgage; NIBSS, Entrazact, PurePay; payroll, shareholder, cards). The PPTX is a 16-slide version of the same content. The PDF (WPS Writer) has no extractable text and is assumed to be an export of the DOCX |
| `Portfolio_At_Risk_Report (79) 2.xlsx` | Loan-level PAR (887 loans × 68 columns) | **Pettysave Microfinance Bank Ltd**, 30-Sep-2026 | BankOne export. Contains PII |
| `CBN REPORT .xlsx` | CBN monthly MFB returns (MMFBR 300, 1000, 001, 141, 201, 202, 221, 311, 312, 321, 322, 451, 501, 641, 642, 651, 711, 746, 761, 762, 763, 764, 771, 811, 933, 951, 980, 996, AML/KYC) | Pettysave MFB (CBN code 51452), 30-Sep-2026 | |
| `Balance_Sheet (61) 2.xlsx` | "Balance Sheet Report 2.0" | Pettysave MFB, 30-Sep-2026 | |
| `Profit_And_Loss_Report … 2.xlsx` | P&L, titled "For the year ended 30-Sep-2026" | Pettysave MFB | Shows a "BANKONE SUBSCRIPTION EXPENSE" line, which identifies the vendor CBS as **BankOne** |
| `CallOver_Report (65).xlsx` | Call-over (posting audit) report | Pettysave MFB, 12-Oct-2026 | 4 entries |
| `TRIAL BALANCE REPORT.xlsx` | Trial balance | **Unnamed entity**, 02-Jul-2026 | A different CoA, capital structure and product set from Pettysave. Also a BankOne export ("Subscription Expense (Bankone)") |

**Context this adds to the blueprint.** Until now the blueprint covered Veend's lending platform (veend-tenant on LendLot/Fineract) and Giro's fixed deposits. These files show a **licensed microfinance bank (Pettysave) running BankOne**, with deposit-taking (demand, savings, term, target savings), teller and vault operations, branches, payroll lending (Lagos State, Federal MDAs/IPPIS, FRSC), and CBN regulatory returns. Nothing in `veend-tenant` or `bill-payment-portal` references Pettysave or BankOne ([V] grep). **Whether Pettysave/BankOne is in scope for Fineract-Mod is an open decision** (§8).

## 2. Findings from the reports (aggregated, verified by re-computation)

### 2.1 The loan book reconciles at loan level

| Check | Result |
|---|---|
| Balance Sheet "Total Loans and Advances" ₦247,801,561.09 − Overdrafts ₦81,523.21 | = **₦247,720,037.88** = Σ PAR "Total Outst. Prin." over 887 loans ✓ |
| BS "Unpaid Micro Loans" + "Unpaid Other Loans" (₦7,742,550.25 + ₦93,365,665.67) | = **₦101,108,215.92** = Σ PAR "Past Due Prin." ✓ |
| PAR "Principal Bal." Σ ₦146,611,821.96 + Past Due Prin. Σ ₦101,108,215.92 | = Total Outst. Prin. ✓. So "Principal Bal." in BankOne means **not-yet-due principal** |
| BS total assets ₦519.74m vs CBN MMFBR 300 ₦503.55m | Difference ₦16.18m = loan-loss provision, a liability on the BS and netted against loans in CBN 300. A presentation difference that reconciles |
| BS deposits ₦222.90m vs CBN 300 ₦227.58m | Difference ₦4.675m = "Mandatory Deposits", carried in other liabilities on the BS. Reconciles |

### 2.2 Loan classification: **the CBN return and the PAR report disagree materially**

| Class (CBN MFB band) | PAR report (BankOne), ₦ | CBN MMFBR 761, ₦'000 |
|---|---|---|
| Performing (0 DPD) | 63,998,826.88 (161 loans) | 199,798 |
| Pass & Watch (1–30) | 52,967,056.18 (142) | 19,477 |
| Substandard (31–60) | 14,770,540.95 (73) | 12,631 |
| Doubtful (61–90) | 12,045,337.79 (80) | 10,424 |
| Lost (91+) | 103,938,276.08 (431) | 5,472 |
| **Total** | 247,720,037.88 (887) | 247,802 |
| PAR>30 (Sub + Doubtful + Lost) | **₦130.75m = 52.8%** | **₦28.53m = 11.5%** |

Notes:
- The PAR report's status and DPD are internally consistent. Every loan's status matches its "Days OverDue" band exactly, and max DPD is 474. The bands match CBN MFB prudential bands, and the report also maps them to IFRS 9 stages (Stage 1 = Performing + Pass & Watch; Stage 2 = Substandard + Doubtful; Stage 3 = Lost).
- **The booked provision follows the CBN 761 figures exactly:**
  - specific provision = 5% × 19,477 + 20% × 12,631 + 50% × 10,424 + 100% × 5,472 = **₦14,184.05k** (equals MMFBR 300 line 10770);
  - general provision = 1% × 199,798 = **₦1,997.98k** (line 10780).
- **If the PAR report's classification is the correct basis**, the same rates give a required provision of about ₦116.20m (specific ₦115.56m + general ₦0.64m) against ₦16.18m booked. That is a shortfall of about **₦100.0m**, which would turn shareholders' funds (₦98.26m) negative.
- [A] We cannot tell from the files which basis is right. The 761 amounts may be **past-due amounts** rather than total exposure, or a different cut-off. This must go to Pettysave's finance and compliance teams. It is exactly the failure mode the recovery team's guide warns about (§7 "Reporting without discrepancies"; §8 "Do not substitute total principal for overdue amount").

### 2.3 Other inconsistencies

| # | Observation | Evidence |
|---|---|---|
| I-1 | Loan **counts** disagree across returns: PAR 887 loans; MMFBR 711 shows 882 individual micro loans; MMFBR 763 maturity profile totals 722 accounts | CBN sheets 711, 763 |
| I-2 | **The P&L period label is suspect.** The BankOne P&L "for the year ended 30-Sep-2026" shows total income ₦13.34m and a deficit of ₦2.15m. CBN MMFBR 1000 for the same date shows interest income ₦122.64m and a loss of ₦101.74m, which equals the BS "Unaudited Loss To Date". The BankOne report probably covers a month, or CBN is cumulative since inception | P&L vs CBN 1000 vs BS |
| I-3 | **737 of 887** loans have a **negative linked customer-account balance** ("Customer Account Bal."), yet BS overdrafts total only ₦81,523. [A] Loan debits may be driving linked savings accounts into unrecognised overdraft | PAR col L |
| I-4 | **The trial balance (unnamed entity, 02-Jul-2026) does not balance as exported**: Σ debit line items ₦1,281,329,039.40 vs Σ credit ₦1,299,813,274.44 (difference ₦18.48m). Its printed "Total Income" (₦1,635.58m) also disagrees with its income lines (₦207.11m), and 14 interest-income GL codes have no names. [A] This may be an export artefact; regenerate it from BankOne | TB file |
| I-5 | Unpaid interest per PAR (₦23.01m) vs BS "Interest In Suspense" (₦20.60m) + "Accrued Interest Receivable" (₦4.12m): the relationship is undocumented | PAR col BP; BS |
| I-6 | The call-over report shows two vault→teller movements with both legs balanced per reference (C102/D102), including teller and client IP. **The approval column is empty** | CallOver |
| I-7 | The PAR "Repayment mode" is blank for 292 loans (₦41.8m) | PAR col AF |

## 3. Requirement traceability: recovery-team guide vs this blueprint

| Guide section | Requirement | Blueprint coverage | Status |
|---|---|---|---|
| §1, §9 | CBS is the system of record; external systems feed it through controlled interfaces | [overview §2](../04-target-architecture/overview.md), [integrations-payments](../05-modules/integrations-payments.md) | ✅ Covered |
| §2 | Customer & KYC → products → transaction engine → GL → reconciliation | Module map ([overview §3](../04-target-architecture/overview.md#3-module-map-and-boundaries)) | ✅ |
| §3.1 | CoA, GL mapping, debit/credit rules, limits, maker-checker, period controls, reversal permissions; failed posting must not update one side | [accounting](../05-modules/accounting.md), [ADR-004](../04-target-architecture/adr/ADR-004-ledger-design.md), [ADR-005](../04-target-architecture/adr/ADR-005-command-pipeline.md) | ✅. Transaction **limits** added (§4 below) |
| §3.2 | Product rules, allocation order, early settlement, version control ("must not silently rewrite contracts") | [loan-products](../05-modules/loan-products.md) BR-LP-05 | ✅ |
| §3.3 | Separate traceable fields: approved, disbursed, repaid, outstanding, accrued, overdue, maturity, DPD | [loan-lifecycle](../05-modules/loan-lifecycle.md), [cob-jobs](../05-modules/cob-jobs.md) | ✅. **Definitions made explicit** (§4) |
| §3.4 | Statements with opening, running and closing balance, value date, reference; equal to the ledger | Partial (loan transactions list only) | ➕ **Added**: statements requirement (reporting) |
| §3.5, §7 | Report families incl. CBN, credit bureau, call-over; as-of date, refresh time, definitions; historical reproducibility | [reporting](../05-modules/reporting.md) | ➕ **Added**: CBN MMFBR catalogue, call-over, report metadata, as-of reproducibility |
| §3.6, §6 | Payment lifecycle (instruction → validation → provider status → settlement → posting → allocation → reconciliation); idempotency; pending/failed traceable | Idempotency and external refs ✅; the pending/failed lifecycle lives in veend-tenant/Giro | ➕ **Added**: reconciliation and exception module (receipt register) |
| §3.7 | Payroll, shareholder, cards, savings/FD, branches | FD ✅ ([fixed-deposits](../05-modules/fixed-deposits.md)); branches partial ([organisation](../05-modules/organisation.md)); payroll/shareholder/cards are not CBS-core | ❓ Scope decision (§8) |
| §4 | Initiation → validation → atomic posting → reconciliation → reporting | [processing §1](../04-target-architecture/processing.md#1-command-pipeline) | ✅ |
| §5 | Repayment posting example (clear receivable if accrued) | [processing §3](../04-target-architecture/processing.md#3-accounting-engine) posting matrix | ✅ Matches |
| §8 | Recovery data model (identifiers, facility, balances, repayment, risk, recovery) and definitions (total outstanding, default amount, DPD, PAR, NPL) | Partial | ➕ **Added**: definitions glossary and a recovery read model |
| §9 | Recovery staff cannot change balances; analytics read-only views | RBAC ✅; read replica ✅ | ➕ **Added**: read-only reporting views for Metabase |
| §10 | Daily and period-end reconciliation controls (7 controls) | Invariants ✅; TB check ✅ | ➕ **Added**: full control catalogue in the reconciliation module |
| §11 | Phased roadmap with exit criteria | [roadmap](../06-roadmap.md) | ✅ Aligned (their Foundation→Lending→Integration→Reporting = our P0–P5) |
| §12 | Vendor-readiness questions | — | ➕ **Answered for Fineract-Mod** (§5 below) |
| §13 | UAT scenarios with evidence | Golden/contract tests | ➕ **Added** as UAT gate (roadmap) |

## 4. Requirements adopted into the blueprint

| ID | Requirement | Where |
|---|---|---|
| RT-01 | **CBN MFB prudential classification** (Performing 0; Pass & Watch 1–30; Substandard 31–60; Doubtful 61–90; Lost ≥91 DPD) as the default delinquency bucket set, with an **IFRS 9 stage** mapping (1: 0–30, 2: 31–90, 3: ≥91) | [cob-jobs](../05-modules/cob-jobs.md), [writeoff-provisioning](../05-modules/writeoff-provisioning.md) |
| RT-02 | **Prudential provisioning** default: specific 5% / 20% / 50% / 100% on P&W / Substandard / Doubtful / Lost, plus general 1% on Performing. Verified against Pettysave's booked figures (§2.2). The **exposure basis** (total outstanding vs past-due) is an explicit, documented parameter | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) |
| RT-03 | **Interest in suspense** for NPL: interest due on loans in Substandard or worse is not recognised as income but held in an Interest-in-Suspense liability | [writeoff-provisioning](../05-modules/writeoff-provisioning.md), [cob-jobs](../05-modules/cob-jobs.md) |
| RT-04 | **Loan balance definitions** exposed with these exact names: `principalNotYetDue`, `principalPastDue`, `totalPrincipalOutstanding`, `interestPastDue`, `feesUnpaid`, `totalOutstanding`, `arrearsAmount` (= past-due principal + interest + fees + penalties), `daysPastDue`, `classification`, `eclStage` | [loan-lifecycle](../05-modules/loan-lifecycle.md) |
| RT-05 | **Customer and loan statements**: opening balance, value date, booking date, reference, description, debit, credit, running balance, closing balance. Closing balance = ledger balance at the same cut-off (tested) | [reporting](../05-modules/reporting.md) |
| RT-06 | **Every report carries metadata**: as-of date, generated-at, data refresh time, definition version, exclusions. Historical reruns are reproducible for a cut-off date | [reporting](../05-modules/reporting.md) |
| RT-07 | **Call-over (posting audit) report**: per business date, every posting with maker, checker, terminal/IP, entry code, reference, narration, approval status | [reporting](../05-modules/reporting.md) |
| RT-08 | **CBN MFB returns** (MMFBR family) generated from the GL plus classification. MVP covers 300, 1000, 761, 763, 771 and 201 if deposits are in scope; the remainder follow | [reporting](../05-modules/reporting.md) |
| RT-09 | **Reconciliation and exception management**: a receipt/settlement register per payment type, daily matching, assigned exceptions (never silent adjustments), and corrective journals linked to the exception | **New**: [reconciliation-exceptions](../05-modules/reconciliation-exceptions.md) |
| RT-10 | **Transaction limits** per role and per command (amount ceilings), in addition to maker-checker thresholds | [platform-core](../05-modules/platform-core.md) |
| RT-11 | **Read-only reporting views** (schema `rpt_views`) for Metabase, with a documented definitions catalogue. No write access | [operations](../04-target-architecture/operations.md) |
| RT-12 | **UAT scenario pack** (guide §13) as the go-live gate per tenant, with evidence retained | [roadmap](../06-roadmap.md), [migration](../07-migration-strategy.md) |

## 5. Vendor-readiness questions answered for Fineract-Mod

| Question (guide §12) | Fineract-Mod answer | Evidence in blueprint |
|---|---|---|
| Does every financial transaction automatically generate balanced entries? | Yes. Every financial command calls `PostingService`, and a deferred constraint rejects unbalanced entries at commit | ADR-004; accounting BR-ACC-01 |
| Are sub-ledger and GL updated atomically? | Yes. One command is one DB transaction covering loan, schedule, transaction, journal, audit and outbox | processing §1 R1 |
| Can principal, accrued interest, arrears, DPD, PAR and NPL be reproduced from documented rules? | Yes. Pure schedule and allocation engines, a documented COB classifier, and RT-04 definitions; golden-tested against Fineract | schedule engine; cob-jobs; ADR-009 |
| How are restructuring, rescheduling, early settlement and write-offs processed while preserving history? | Immutable transactions with reversal/replay. Write-off and recovery are in MVP; rescheduling is in scope (P7-R; history kept in `loan_installment_history`) | loan-transactions; writeoff-provisioning; restructuring-rescheduling |
| Documented APIs, unique references, duplicate prevention? | Yes. OpenAPI from zod, `Idempotency-Key`, and `UNIQUE(paymentType, externalRef)` | api-design §2; BR-TXN-01 |
| How are NIBSS, transfers, providers and mandates reconciled? | Per-rail clearing accounts plus the receipt register and daily matching (RT-09). Rails themselves remain in veend-tenant/Giro | reconciliation-exceptions |
| Can pending, failed, delayed and reversed payments be tracked? | Reversed: yes (CBA). Pending and failed: in the receipt register fed by the channel systems (RT-09) | reconciliation-exceptions |
| Can reports be reproduced for a historical as-of date? | Yes. COB snapshots plus immutable ledger; report metadata (RT-06) | reporting |
| Maker-checker, RBAC, audit logs, backups and DR tested? | Designed (platform-core, operations); tested at gate G-SEC and quarterly restore drills | platform-core; operations §3 |
| How are CBN and credit-bureau definitions validated? | CBN returns are versioned report modules with sign-off from compliance (RT-08); bureau extracts are P7-G | reporting |

## 6. What does **not** change

- **The architecture holds.** The guide's principles (single system of record, atomic balanced postings, idempotent integrations, reconciled reporting) are already the blueprint's core design decisions.
- Build-vs-buy: the guide is written for configuring a **vendor** CBS (BankOne-style menus). The blueprint's decision (ADR-001: build Fineract-Mod) stands unless leadership decides to standardise on a vendor. [A] The guide's existence suggests that option may be under discussion; confirm (§8).

## 7. Data-handling actions

- `docs/cba-notes-recovery-team/` contains BVNs, phone numbers, addresses and guarantor details for 887 borrowers. **It is now listed in `.gitignore`** (2026-11-02) and is not tracked. Recommend also moving it to a restricted share.
- Nothing from these files has been copied into the blueprint except aggregates and report structures.

## 8. Open decisions raised by this submission

| # | Decision | Why it matters |
|---|---|---|
| 1 | ✅ **Decided 2026-11-02: Fineract-Mod must have all features in the recovery-team documents** ([ADR-010](../04-target-architecture/adr/ADR-010-full-mfb-core-banking-scope.md), [feature register](../03-gap-analysis/recovery-team-feature-register.md)). Pettysave/BankOne migration is planned as P8-B. Originally asked: **Is Pettysave MFB (on BankOne) in scope for Fineract-Mod?** If yes, scope grows to full deposit-taking (demand/current, savings, target savings, term deposits), teller/vault/branch cash, NIBSS transfers, and the full CBN MMFBR return set | It would add phases and change the MVP. Current-account and teller modules are not specified today |
| 2 | **Which entity does the trial balance belong to?** | It is not Pettysave, and it does not balance as exported |
| 3 | **Which classification basis is correct for Pettysave's CBN 761 and provisioning?** | A potential ~₦100m provision shortfall (§2.2); a regulatory issue independent of Fineract-Mod |
| 4 | **Build vs buy:** is the guide input to configuring BankOne, or to Fineract-Mod? | ADR-001 |
