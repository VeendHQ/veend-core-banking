# Gap Analysis: Capability Matrix

Each capability from brief §4.3 is classified using the brief's five categories. A capability can belong to more than one category.

| Code | Meaning |
|---|---|
| **1** | Already available in Apache Fineract |
| **2** | Currently implemented in veend-tenant (our lending platform) |
| **3** | Required but not currently implemented anywhere we use |
| **4** | Available, but requires modification for Fineract-Mod |
| **5** | Not required for the initial release; relevant to future expansion |

**Used today** means veend-tenant actually exercises it through LendLot ([V] by the [endpoint inventory](../02-current-state/lendlot-endpoint-inventory.md)).
**Fineract-Mod scope:** MVP = phases P0–P6; Future = P7+.

| # | Capability | Cat. | Fineract evidence | veend-tenant evidence | Used today? | Fineract-Mod scope and spec |
|---|---|---|---|---|---|---|
| 1 | **Client and customer management** | 1, 2, 4 | `m_client`, `ClientsApiResource`, datatables | `createBorrower`, `updateBorrower`; KYC, BVN and bank data held locally in `Users` | Yes (create, search by phone, update name/phone/email) | MVP. KYC fields (BVN hash, NIN) become first-class instead of datatable data → [clients](../05-modules/clients.md) |
| 2 | **Group management** | 1, 5 | `m_group`, GLIM | none | No | Future → [future/groups](../05-modules/future/groups.md) |
| 3 | **Loan products and configuration** | 1, 2, 4 | `m_product_loan` + about 10 satellite tables | `createLoanProduct` with hard-coded flat/accrual-upfront/GL ids; local `Loanproducts` mirror | Yes | MVP, with GL roles configurable per product → [loan-products](../05-modules/loan-products.md) |
| 4 | **Loan application and approval workflow** | 1, 2, 4 | submit (100) → approve (200), `undoapproval`, `reject`, `withdrawnByApplicant`; maker-checker | Auto-approve immediately after create (`acceptTerms`) | Yes (no human approval) | MVP. Approval policies (auto, or maker-checker above a threshold) → [loan-lifecycle](../05-modules/loan-lifecycle.md) |
| 5 | **Loan disbursement** | 1, 2, 4 | `disburse`, `undodisbursal`, tranche `editDisbursements`, disburse-to-savings | Money moved via Giro/Remita, then CBA `disburse` dated today | Yes | MVP, value-dated, with net-of-fee disbursement as an explicit product feature → [loan-lifecycle](../05-modules/loan-lifecycle.md) |
| 6 | **Loan repayment and allocation** | 1, 2, 4 | `repayment` command; 9 classic strategies + advanced allocation | All channels → `makeRepayment`, `paymentTypeId:1`, date today | Yes | MVP: idempotent, value-dated, per-channel payment types; start with `mifos-standard-strategy` plus the strategy used by live products [A] → [loan-transactions](../05-modules/loan-transactions.md) |
| 7 | **Loan restructuring and rescheduling** | 1, 3, 5 | `m_loan_reschedule_request`, `reAge`, `reAmortize` | Mandate "restructuring" only (`restructuredRemitaMandateReferences`), not in the CBA | No | Future (P7). [A] Ask the business whether restructures are done off-system today → [future/rescheduling](../05-modules/future/rescheduling.md) |
| 8 | **Savings and deposit accounts** | 1, 2, 5 | `fineract-savings` | Giro (`bill-payment-portal`) opens one Fineract savings account per wallet account. Balances were migrated to Giro's own SQL ledger ("V3"); Fineract is still used for liens ([V] D1.4 §2) | Partly (wallet mirror, liens) | Future, pending decision on whether the Giro wallet ledger moves → [future/savings](../05-modules/future/savings.md) |
| 9a | **Fixed deposits** | 1, 2, 3, 4 | `fineract-savings` FD products and accounts; savings accounting roles | Giro "loan subscriptions": Fineract FD savings accounts opened, but current (G2) deposits and interest are tracked **off-ledger** (`Primelogs`, DOUBLE balances, on-the-fly interest) ([V] D1.4 §3, FD-01) | **Yes** (Giro) | **MVP (P5-D)** → [fixed-deposits](../05-modules/fixed-deposits.md) |
| 9b | **Recurring deposits**, FD rollover and compounding | 1, 5 | `fineract-savings` (RD) | none | No | Future → [future/deposits](../05-modules/future/deposits.md) |
| 10 | **General ledger and accounting** | 1, 4 | `fineract-accounting`, `acc_*` | Implicit only (GL ids hard-coded in the product) | Yes (automatic) | MVP, the core of the build → [accounting](../05-modules/accounting.md) |
| 11 | **Chart of accounts** | 1, 3 | `acc_gl_account` (header/detail, types 1–5) | Not managed; assumed seeded by tenant-creator [A] | Indirect | MVP: CoA template per tenant, seeded from the migrated LendLot CoA → [accounting](../05-modules/accounting.md) |
| 12 | **Journal entries** | 1, 4 | `acc_gl_journal_entry` (single-sided rows), manual JEs via `/journalentries` | none | Indirect | MVP: double-entry header plus lines, immutable, with manual JE under maker-checker → [accounting](../05-modules/accounting.md) |
| 13 | **Financial transactions (generic)** | 1, 4 | `m_loan_transaction`, `m_payment_detail` | `RepaymentLogs`, `Webhooks`, `APICalls` | Yes | MVP: external_ref and idempotency on every transaction → [loan-transactions](../05-modules/loan-transactions.md) |
| 14 | **Interest calculation and accrual** | 1, 4 | Flat/declining generators, EMI calculator, `AddPeriodicAccrualEntries`, accrual-upfront | Flat only, ACCRUAL_UPFRONT; schedules taken from CBA output | Yes | MVP: flat + declining (EMI); accrual upfront + periodic; golden-tested → [loan-schedule-engine](../05-modules/loan-schedule-engine.md), [cob-jobs](../05-modules/cob-jobs.md) |
| 15 | **Fees, charges and penalties** | 1, 2, 4 | `m_charge` (time types 1–16, calc types 1–5), overdue penalties via COB | Charge CRUD (`chargeTimeType` default 8, calc 2); no penalty automation | Yes (fees); No (penalties) | MVP: disbursement, installment and specified-due-date fees; overdue penalty step → [charges](../05-modules/charges.md) |
| 16 | **Portfolio management** | 1, 2, 4 | Loan officer, delinquency buckets, arrears aging | Recovery agents, defaults, `Cbaloanarrears` snapshot | Yes (reports) | MVP: delinquency buckets and arrears in the CBA; recovery workflow stays in veend-tenant → [cob-jobs](../05-modules/cob-jobs.md), [reporting](../05-modules/reporting.md) |
| 17 | **Accounting rules** | 1, 4 | `AccountingRuleType` 1–4, `acc_accounting_rule` (manual JE templates) | Always 4 (ACCRUAL_UPFRONT) | Yes | MVP: NONE, CASH, ACCRUAL_UPFRONT, ACCRUAL_PERIODIC → [accounting](../05-modules/accounting.md) |
| 18 | **Financial reporting** | 1, 2, 3 | Stretchy reports (Trial Balance, GL, Active Loans, Aging, PAR) | Dashboards built on 2 reports + local metrics | Yes (2 reports) | MVP: Trial balance, GL detail, portfolio summary, aging, PAR, disbursement and collection registers, plus compat reports → [reporting](../05-modules/reporting.md) |
| 19 | **Provisioning and write-offs** | 1, 2, 3, 4 | `writeoff`, `undowriteoff`, `recoverypayment`, `chargeOff`, provisioning criteria/entries | Local `isWriteOff` flag only | **No** (gap DF-04) | MVP: write-off, recovery and charge-off; provisioning (criteria + monthly entries) → [writeoff-provisioning](../05-modules/writeoff-provisioning.md) |
| 20 | **Reversals and adjustments** | 1, 2, 4 | `undo` / adjust transaction, reverse-replay of later transactions, reversed JEs | `reverseRepayment` (match by amount) | Yes | MVP: reverse by id or external ref, with replay of subsequent allocations → [loan-transactions](../05-modules/loan-transactions.md) |
| 21 | **Audit trails** | 1, 2, 4 | `m_portfolio_command_source` (every command, maker/checker, JSON) | `Audittrails` model, `logAudit` job, `APICalls` | Partial | MVP: synchronous, append-only, hash-chained command audit → [platform-core](../05-modules/platform-core.md) |
| 22 | **Maker-checker authorization** | 1, 3 | Per-permission enablement, checker superuser | none | No | MVP: configurable per command plus amount thresholds → [platform-core](../05-modules/platform-core.md) |
| 23 | **User, role and permission management** | 1, 2, 4 | `m_appuser`, `m_role`, `m_permission` | veend-tenant has its own dashboard users and roles; CBA used via one service account | Indirect | MVP: CBA users for back-office plus OAuth2 service clients for veend-tenant → [platform-core](../05-modules/platform-core.md) |
| 24 | **Business dates and end-of-day processing** | 1, 3, 4 | `m_business_date`, Loan COB, inline COB, job scheduler | Clock cron jobs (local) | No (in CBA) [A] confirm whether LendLot runs COB jobs | MVP: business date + COB pipeline → [cob-jobs](../05-modules/cob-jobs.md) |
| 25 | **Notifications and integrations** | 1, 2, 4 | External events (Avro over Kafka/JMS), webhooks (`m_hook`), SMS/email campaigns | All notifications in veend-tenant (Termii SMS, mail); rails integrated in veend-tenant | Not from CBA | MVP: transactional outbox → signed webhooks to veend-tenant; rails stay in veend-tenant → [events-notifications](../05-modules/events-notifications.md), [integrations-payments](../05-modules/integrations-payments.md) |
| 26 | Offices, staff, holidays, working days | 1, 4 | `m_office`, `m_staff`, `m_holiday`, `m_working_days` | Office hard-coded to 1; staff/office methods dead | Minimal | MVP (minimal): head office plus optional branches; holidays affect due dates → [organisation](../05-modules/organisation.md) |
| 27 | Custom fields (datatables) | 1, 2, 4 | `x_registered_table` + runtime DDL | `userdata.email`, `loandata.{loanReason, channel, remitaMandateReference}` | Yes | MVP: typed JSONB `attributes` with JSON-schema per entity, no runtime DDL → [extensibility-datatables](../05-modules/extensibility-datatables.md) |
| 28 | Top-up loans | 1, 2, 3 | `m_loan_topup`, `isTopup` + `loanIdToClose` | Gate local; CBA top-up disabled (DF-07) | No | MVP (behind a flag): atomic top-up → [loan-lifecycle](../05-modules/loan-lifecycle.md) |
| 29 | Loan sale / participation (on-lending) | 1 (investor), 2, 4, 5 | `fineract-investor` (external asset owner transfers) | Cross-tenant duplicate loans (F5). FD investor money funds on-lending via Giro → veend-tenant `/onlending/disburse` → loan to "Eden" ([V] D1.4 §3.2) | Yes (as duplicates) | P7 for full participation accounting. The **deployment record** linking a deposit to an on-lending loan is MVP (fixed-deposits FR-FD-11) → [future/investor](../05-modules/future/investor.md) |
| 30 | Credit bureau / decisioning | 1 (credit bureau module, limited), 2 | `CreditBureau*` | Mono, Indicina, credit check, BVN in veend-tenant | Not via CBA | Stays in veend-tenant. Out of CBA scope [R] |
| 31 | Collections mandates (Remita/OnePipe/Vida) | 2 | — | veend-tenant | n/a | Stays in veend-tenant. CBA exposes schedules and events |
| 32 | Insurance claims | 2, 3 | — | Local flag only | No | MVP-lite: post the insurance recovery as a `recoverypayment` with a dedicated payment type → [writeoff-provisioning](../05-modules/writeoff-provisioning.md) |
| 33 | Multi-currency | 1, 5 | `m_currency`, per-product currency | NGN hard-coded | No | Schema supports currency from day one; only NGN enabled (P7 to enable more) |
| 34 | Tax on fees (VAT) | 1, 5 | `fineract-tax` | none | No | Future [A]: check the Nigerian VAT treatment of loan fees with finance |

## Summary

- **Parity (must exist at cut-over):** items 1, 3–6, 13, 14, 15 (fees), 20, 25 (webhooks), 27, plus the two compat reports (18). All are listed in the [endpoint inventory](../02-current-state/lendlot-endpoint-inventory.md).
- **Correctness upgrades in MVP** (missing or wrong today): 10–12 explicit GL, 17, 19 write-off and provisioning, 21–24 governance and COB, 28 top-up, value-dating and idempotency.
- **Fixed deposits (9a), parity plus ledger for Giro:** the Giro savings compat endpoints (D2.7 §4), plus booking the deposit liability and interest expense that are off-ledger today.
- **Future:** 2, 7, 8, 9b, 29 (beyond the deployment record), 33, 34.
- **Out of CBA scope** (stays in veend-tenant): 30, 31, borrower-facing notifications, eligibility and decisioning.
