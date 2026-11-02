# Recovery-Team Feature Register (authoritative scope, ADR-010)

Every feature named in the recovery-team documents (`VeendHQ_Core_Banking_Application_Blueprint` DOCX/PPTX, plus the fields and report types in the Pettysave BankOne exports) mapped to the Fineract-Mod module that delivers it and its roadmap phase. **This register is the acceptance checklist for "all features stated in the recovery team's documents."**

Source refs: §n = guide section; PAR = Portfolio-at-Risk report columns; CBN = MMFBR returns; CO = call-over report; TB/BS/PL = financial reports.

| # | Feature (as named by the recovery team) | Source | Module | Phase |
|---|---|---|---|---|
| **Customer & KYC** |||||
| F-01 | Customer ID, BVN/NIN, contact details | §2 | [clients](../05-modules/clients.md) | P1 |
| F-02 | Customer risk profile (KYC tier, risk rating, PEP flag) | §2 | [clients](../05-modules/clients.md) FR-CLI-09..11 | P1 |
| **Accounting Management** |||||
| F-03 | GL accounts; structured chart of accounts | §3.1 | [accounting](../05-modules/accounting.md) | P3 |
| F-04 | Account categories | §3.1 | [accounting](../05-modules/accounting.md) FR-ACC-10 | P3 |
| F-05 | Fixed assets | §3.1 | [fixed-assets-amortization](../05-modules/fixed-assets-amortization.md) | P7-E |
| F-06 | Amortization (prepayments, intangibles, deferred fees) | §3.1, BS | [fixed-assets-amortization](../05-modules/fixed-assets-amortization.md) | P7-E |
| F-07 | Product/transaction → GL mapping; Dr/Cr rules | §3.1 | [accounting](../05-modules/accounting.md), [processing §3](../04-target-architecture/processing.md#3-accounting-engine) | P3 |
| F-08 | Transaction limits; maker-checker; period controls; reversal permissions | §3.1 | [platform-core](../05-modules/platform-core.md) FR-PLT-06, 11; [accounting](../05-modules/accounting.md) closures | P0–P3 |
| **Posting Management** |||||
| F-09 | Account posting (manual single/multi-leg) | §3.1 | [posting-management](../05-modules/posting-management.md) | P6-B |
| F-10 | Batch uploads | §3.1 | [posting-management](../05-modules/posting-management.md) | P6-B |
| F-11 | Cash posting (teller deposits/withdrawals) | §3.1, CO | [teller-branch-cash](../05-modules/teller-branch-cash.md) | P6-B |
| F-12 | Funds transfers (internal, account to account) | §3.1 | [posting-management](../05-modules/posting-management.md) | P6-B |
| F-13 | Loan repayment posting (manual/bulk) | §3.1 | [posting-management](../05-modules/posting-management.md) → [loan-transactions](../05-modules/loan-transactions.md) | P6-B |
| F-14 | Posting approvals | §3.1 | [posting-management](../05-modules/posting-management.md) + maker-checker | P6-B |
| F-15 | Reversals | §3.1 | [posting-management](../05-modules/posting-management.md), [loan-transactions](../05-modules/loan-transactions.md) | P3/P6-B |
| **Product Management** |||||
| F-16 | Fixed deposits | §3.2 | [fixed-deposits](../05-modules/fixed-deposits.md) | P5-D |
| F-17 | Savings accounts | §3.2 | [savings-current-accounts](../05-modules/savings-current-accounts.md) | P6-A |
| F-18 | Current (demand) accounts | §3.2, BS | [savings-current-accounts](../05-modules/savings-current-accounts.md) | P6-A |
| F-19 | Commitment savings (target savings) | §3.2, TB | [savings-current-accounts](../05-modules/savings-current-accounts.md) | P6-A |
| F-20 | Loan products | §3.2 | [loan-products](../05-modules/loan-products.md) | P2 |
| F-21 | Quick loan product creation | §3.2 | [loan-products](../05-modules/loan-products.md) FR-LP-08 | P2 |
| F-22 | Mortgage loan products | §3.2 | [loan-products](../05-modules/loan-products.md) FR-LP-09 + [collateral-guarantors](../05-modules/collateral-guarantors.md) | P7-R |
| F-23 | Interest rate and calculation method; tenor, frequency, maturity, grace | §3.2 | [loan-products](../05-modules/loan-products.md), [schedule engine](../05-modules/loan-schedule-engine.md) | P2 |
| F-24 | Repayment allocation order | §3.2 | [loan-transactions](../05-modules/loan-transactions.md) | P3 |
| F-25 | Early settlement | §3.2 | [loan-transactions](../05-modules/loan-transactions.md) (payoff/foreclosure) | P3 |
| F-26 | Charges and penalties | §3.2 | [charges](../05-modules/charges.md) | P2/P4 |
| F-27 | Collateral requirements | §3.2, PAR | [collateral-guarantors](../05-modules/collateral-guarantors.md) | P7-R |
| F-28 | Restructuring, rescheduling (and refinancing) | §3.2, §3.3, PAR | [restructuring-rescheduling](../05-modules/restructuring-rescheduling.md) | P7-R |
| F-29 | Write-off and impairment treatment | §3.2 | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) | P4 |
| F-30 | Product version control | §3.2 | [loan-products](../05-modules/loan-products.md) FR-LP-04 | P2 |
| **Loan Portfolio Management** |||||
| F-31 | Provision billing (periodic provisioning run and posting) | §3.3 | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) | P4 |
| F-32 | Provision loans (per-loan provision) | §3.3 | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) | P4 |
| F-33 | Provision overdrawn accounts | §3.3, BS | [savings-current-accounts](../05-modules/savings-current-accounts.md) (overdraft classification) + [writeoff-provisioning](../05-modules/writeoff-provisioning.md) | P6-A |
| F-34 | Loan application and **appraisal**, approval, disbursement | §3.3 | [loan-lifecycle](../05-modules/loan-lifecycle.md) FR-LN-14 | P3 |
| F-35 | Schedule, accruals, allocation; balance fields; DPD, arrears, PAR, NPL | §3.3 | schedule engine, cob-jobs, loan-lifecycle FR-LN-12 | P2–P4 |
| F-36 | Recoveries (post write-off) | §3.3 | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) | P4 |
| F-37 | Impairment / **expected credit loss** calculation | §3.3 | [writeoff-provisioning](../05-modules/writeoff-provisioning.md) FR-WO-09 | P7-G |
| **Customer Account Statement Management** |||||
| F-38 | Account statements; loan account statements | §3.4 | [reporting](../05-modules/reporting.md) FR-RPT-09 | P5 |
| F-39 | Multiple statements; batch statements; view batch statements | §3.4 | [reporting](../05-modules/reporting.md) FR-RPT-13 | P6-A |
| **Reports Management** |||||
| F-40 | Loan account reports | §3.5 | [reporting](../05-modules/reporting.md) | P5 |
| F-41 | GL account reports | §3.5 | [accounting](../05-modules/accounting.md), reporting | P3/P5 |
| F-42 | Financial reports: TB, balance sheet, P&L, **cash flow** | §3.5 | [reporting](../05-modules/reporting.md) | P5 |
| F-43 | Management reports (portfolio, collections, PAR/NPL, income, recovery, maturity pipeline) | §3.5, §7 | [reporting](../05-modules/reporting.md) | P5 |
| F-44 | CBN reports (full MMFBR set + AML/KYC) | §3.5, CBN | [regulatory-reporting](../05-modules/regulatory-reporting.md) | P7-G |
| F-45 | Credit bureau reports | §3.5, §9 | [regulatory-reporting](../05-modules/regulatory-reporting.md) | P7-G |
| F-46 | Activity / call-over reports | §3.5, CO | [reporting](../05-modules/reporting.md) FR-RPT-10 | P5 |
| **Bills Payments and Transfers** |||||
| F-47 | Bills payment | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) | P6-C |
| F-48 | Transfers (outbound interbank) | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) | P6-C |
| F-49 | NIBSS management | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) | P6-C |
| F-50 | NIBSS institution management | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) | P6-C |
| F-51 | Entrazact management | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) (provider adapter) [A] | P6-C |
| F-52 | PurePay management | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) (provider adapter) [A] | P6-C |
| F-53 | Payment approvals | §3.6 | [payments-transfers-bills](../05-modules/payments-transfers-bills.md) + maker-checker | P6-C |
| F-54 | Inbound repayment flow (8 steps); pending/failed/reversed traceability | §3.6, §6 | [reconciliation-exceptions](../05-modules/reconciliation-exceptions.md), payments | P5/P6-C |
| **Payroll, Shareholder, Cards, Branch** |||||
| F-55 | Payroll (journals to GL; salaries, deductions, taxes, liabilities, bank payments) | §3.7, TB | [payroll](../05-modules/payroll.md) | P7-E |
| F-56 | Shareholder management (shareholding, issuance, dividends) | §3.7, TB/BS | [shareholder-management](../05-modules/shareholder-management.md) | P7-E |
| F-57 | Card management (profiles; card/cash disputes; card reports) | §3.7 | [card-management](../05-modules/card-management.md) | P7-K |
| F-58 | Savings and FD interest; maturity instructions | §3.7 | savings-current-accounts; [fixed-deposits](../05-modules/fixed-deposits.md) FR-FD-16 | P5-D/P6-A |
| F-59 | Branch management: branch access, transaction ownership, branch-to-head-office reconciliation | §3.7, §10 | [teller-branch-cash](../05-modules/teller-branch-cash.md), [organisation](../05-modules/organisation.md) | P6-B |
| F-60 | Teller and vault operations (cash movement, teller overages) | CO, TB | [teller-branch-cash](../05-modules/teller-branch-cash.md) | P6-B |
| **Recovery** |||||
| F-61 | Recovery officer, contact attempts, promises-to-pay, field visits, recovery status | §8, §9 | [recovery-collections](../05-modules/recovery-collections.md) | P7-R |
| F-62 | Guarantors (and guarantor exposure) | PAR | [collateral-guarantors](../05-modules/collateral-guarantors.md) | P7-R |
| F-63 | Account officer, economic sector, lending model, MDA/IPPIS tagging | PAR, CBN 711/762 | [loan-lifecycle](../05-modules/loan-lifecycle.md) FR-LN-15 | P3 |
| **Controls, integration, reporting quality** |||||
| F-64 | Daily and period-end reconciliation controls | §10 | [reconciliation-exceptions](../05-modules/reconciliation-exceptions.md) | P5 |
| F-65 | Documented APIs / file interfaces for payments, bureaus, recovery tools, analytics | §9 | [api-design](../04-target-architecture/api-design.md), regulatory-reporting, recovery-collections, operations §2a | P5+ |
| F-66 | Report metadata and historical reproducibility | §7 | [reporting](../05-modules/reporting.md) FR-RPT-08 | P5 |
| F-67 | UAT scenario pack with evidence | §13 | [roadmap](../06-roadmap.md) | Gates |
