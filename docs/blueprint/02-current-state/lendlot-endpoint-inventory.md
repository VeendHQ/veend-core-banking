# D2.2 LendLot Endpoint Inventory (the compatibility contract)

Every Fineract call veend-tenant can make. Source: `veend-tenant/services/lendlot/index.js` ([V], read in full).

**How to read it.** "Live callers" counts call sites outside `services/lendlot` and `services/loandisk`, found by grep over `controllers/`, `services/queue/`, `helpers/` and `scripts/`. Methods with **0** live callers are dead code. The compat facade does not need to serve them.

## 1. Transport contract (applies to all calls)

| Aspect | Current behaviour | Evidence |
|---|---|---|
| Base URL | `config.lendlotBaseUrl` (per tenant in `Accesstokens.config`), ending in `/fineract-provider`. Paths are `/api/v1/...` | [V] `:62` etc. |
| Auth | `Authorization: Basic base64(user:pass)` | [V] `getHeaders` `:48-56` |
| Tenant | `Fineract-Platform-TenantId: <lendlotTenantId>`; **forced to `veendhqautolending` when `config.env !== 'production'`** | [V] `:45` |
| Content type | JSON. Fineract date params use `dateFormat: 'dd MMMM yyyy'` and `locale: 'en'` | [V] |
| Dates | Always `moment().format('DD MMMM YYYY')`, i.e. **server-local today** | [V] `:500, 726, 1333, 1401, 1509, 1541, 1563, 1614` |
| TLS | `rejectUnauthorized: false`, so certificates are not verified | [V] `services/request/index.js` |
| Replay cache | Before calling, looks up `APICalls{RequestId, service}`. If a prior response had status 200, returns it **without calling** | [V] `services/request/index.js` |
| GET body leak | For GET, `data` (= `this.httpBody`) is sent as the **query string**. Because `httpBody` is shared and mutable, earlier POST fields leak into later GETs | [V] `services/request/index.js` (`options.qs = options.data`) |
| Errors | Rejects with the Fineract error body (`err.response.body`) or `{type:'internal error'}` | [V] |

## 2. Live endpoints (must be served by the Fineract-Mod LendLot-compat facade)

### 2.1 Clients and datatables

| # | Fineract endpoint | Client method | Live callers | Request fields sent | Response fields read |
|---|---|---|---|---|---|
| C1 | `GET /search?exactMatch=false&resource=clients,clientIdentifiers,groups&query={phone}` | `search` (defined twice: `:58`, `:469`) | via `createBorrower` | — | `[].entityType`, `entityId`, `entityMobileNo` |
| C2 | `POST /clients` | `createBorrower` `:488` | 8 (`jobs.js`, `Loans.js`) | `mobileNo, firstname, lastname, officeId (=branch, always 1), active:true, activationDate, legalFormId:1, datatables:[{registeredTableName:'userdata', data:{email}}]` | `clientId` |
| C3 | `GET /clients/{id}` | `getBorrower` `:545` | 1 (`jobs.syncUserFrom`) | — | whole object (mapped by caller) |
| C4 | `PUT /clients/{id}` + `PUT /datatables/userdata/{id}?genericResultSet=true` (fire-and-forget) | `updateBorrower` `:564` | 3 (`jobs.syncUserTo`, `syncUserToLoanDisk`) | `mobileNo, firstname, lastname`; datatable body is `{email, ...entire clientBody}` | `changes`, `clientId` |

### 2.2 Loan products and charges

| # | Endpoint | Method | Live callers | Request | Response read |
|---|---|---|---|---|---|
| P1 | `GET /loanproducts` + per product `GET /loanproducts/{id}?template=true` | `getLoanProducts` `:350` | 1 (`jobs.syncLoanProductFrom`) | — | `id, name, interestRatePerPeriod, min/maxInterestRatePerPeriod, numberOfRepayments, min/maxNumberOfRepayments, principal, min/maxPrincipal, charges[].{chargeCalculationType.value, penalty, amount, minCap, maxCap, feeInterval, feeFrequency.value}` |
| P2 | `GET /loanproducts/template` then `POST /loanproducts` | `createLoanProduct` `:829` | 8 (`Loanproducts.create`, `Loans.js`, on-lending/Vida/IPPIS jobs) | **Hard-coded**: `accountingRule:4` (ACCRUAL_UPFRONT), `interestType:1` (FLAT), `currencyCode:'NGN'`, `digitsAfterDecimal:'2'`, `repaymentEvery:1`, `transactionProcessingStrategyId: template.transactionProcessingStrategyOptions[0].id`, GL ids `fundSource 5, loanPortfolio 9, interestOnLoan 30, incomeFromFee 32, incomeFromPenalty 31, writeOff 24, receivableInterest 6, receivableFee 10, receivablePenalty 7, transfersInSuspense 8, overpaymentLiability 19, incomeFromRecovery 33`; from body: name, shortName (first 3 chars), principal min/default/max, interest min/default/max, repayments min/default/max, `charges` | `resourceId` |
| P3 | `GET /loanproducts/{id}?template=true` then `PUT /loanproducts/{id}` | `updateLoanProduct` `:915` | 1 (`Loanproducts.updateOne`) | Echoes the template with name/principal/interest/tenure ranges and `charges` | `changes` |
| P4 | `POST /charges` | `createCharge` `:1092` | 1 (`Lendlotcharges.create`) | `name ('Loan service fee'), chargeAppliesTo (1=LOAN), currencyCode 'NGN', amount, chargeTimeType (default 8=INSTALMENT_FEE), chargeCalculationType (default 2=% of amount), chargePaymentMode (0=REGULAR), penalty, active, minCap, maxCap, feeFrequency, feeInterval` | `resourceId` |
| P5 | `PUT /charges/{id}` | `updateCharge` `:1135` | 1 | same fields; `amount` is `parseFloat` | `changes` |
| P6 | `DELETE /charges/{id}` | `deleteCharge` `:1177` | 1 | — | `resourceId` |

### 2.3 Loan origination and lifecycle

| # | Endpoint | Method | Live callers | Request | Response read |
|---|---|---|---|---|---|
| L1 | `GET /loans/template?activeOnly=true&clientId&productId&staffInSelectedOfficeOnly=true&templateType=individual` | `getLoanProductsForClient` `:1287` | internal (L2, L3) | — | `termPeriodFrequencyType.id, repaymentEvery, repaymentFrequencyType.id, interestRatePerPeriod, amortizationType.id, interestType.id, interestCalculationPeriodType.id, transactionProcessingStrategyId, charges[].{chargeId, amount}` |
| L2 | `POST /loans?command=calculateLoanSchedule` | `calculateLoan` `:1312` | 6 (`Webhooks.js` USSD, `jobs.getEligibility`) | `productId, principal, loanTermFrequency = numberOfRepayments = tenure, repaymentEvery, repaymentFrequencyType, interestRatePerPeriod (override or template), amortizationType, interestType, interestCalculationPeriodType, expectedDisbursementDate=today, submittedOnDate=today, transactionProcessingStrategyId, loanType:'individual', clientId, charges[{chargeId, amount}]` | `totalOutstanding, totalPrincipalExpected, totalInterestCharged, totalRepaymentExpected, totalFeeChargesCharged, periods[]` |
| L3 | `POST /loans` | `createLoan` `:1371` | 15 (`Users.acceptTerms`, `Webhooks.js` ×5+, `jobs.createLoan`, on-lending/Vida/prime jobs, `workers.js`) | as L2, plus `datatables:[{registeredTableName:'loandata', data:{loanReason, channel}}]`; optional `isTopup, loanIdToClose` (never set in practice; [V] `Users.js:2191-2196`) | `loanId` |
| L4 | `POST /loans/{id}?command=approve` | `approveLoan` `:1501` | 25 | `approvedOnDate=today`, optional `approvedLoanAmount` | (none checked) |
| L5 | `POST /loans/{id}/disbursements/editDisbursements` *(only when `approvedLoanAmount` passed)* then `POST /loans/{id}?command=disburse` | `disburseLoan` `:1528` | 24 | edit: **bug**, sends `this.httpBody` instead of the built `body`; disburse: `actualDisbursementDate=today`, optional `transactionAmount` | (none checked) |
| L6 | `POST /loans/{id}?command=undoApproval` | `undoApproval` `:1582` | 4 (`processDisbursementStatus`, `processOnlendingDisbursementStatus`) | `note` | swallowed: resolves `{isError:true}` on failure |
| L7 | `POST /loans/{id}?command=reject` | `rejectLoan` `:1606` | 4 | `rejectedOnDate=today, note` | swallowed as L6 |
| L8 | `GET /loans/{id}?associations=all` or `=repaymentSchedule` | `getLoan` `:1632` | 30 | — | see §4 |
| L9 | `GET /loans/{id}?associations=repaymentSchedule` | `getLoanSchedule` `:1654` | 12 (`Users.js`, `Dashboard.js`, `jobs.js`) | — | `repaymentSchedule.periods[]` (`period, dueDate, totalDueForPeriod`, …) |
| L10 | `PUT /datatables/loandata/{loanId}?genericResultSet=true` | `updateLoanDatatable` `:629`, `LoanDatatableEntry` `:605` | 11 + 1 | `{remitaMandateReference}`, other loandata columns | (none) |

### 2.4 Transactions

| # | Endpoint | Method | Live callers | Request | Response read |
|---|---|---|---|---|---|
| T1 | `POST /loans/{id}/transactions?command=repayment` | `makeRepayment` `:717` | 6 (`jobs.payoff`, `processRepaymentFromRemita`, `processRepaymentFromIppis`, on-lending repay) | `transactionDate=today`, `transactionAmount`, `paymentTypeId:1` (hard-coded), optional `note`. **RequestId = `this.reference` (caller-supplied, no random suffix)** | `changes` (presence = success). Side effect: mirrors to LoanDisk if `loan.loanDiskId` |
| T2 | `POST /loans/{id}/transactions/{txId}?command=undo` | `undoRepayment` `:775` | 1 (`Loans.reverseRepayment`) | `transactionAmount:0, transactionDate=today` | (none) |

### 2.5 Reports

| # | Endpoint | Method | Live callers | Response contract |
|---|---|---|---|---|
| R1 | `GET /runreports/ActiveLoansSummary?R_currencyId=-1&R_fundId=-1&R_loanOfficerId=-1&R_loanProductId=-1&R_loanPurposeId=-1&R_officeId=1&R_parType=3` | `getPortfolio` `:1695` | 1 (`Dashboard.portfolio`) | `{columnHeaders[], data[{row[]}]}`. Read **positionally**; see §3 |
| R2 | same with `R_currencyId=NGN&R_parType=4` | `getActiveLoansSummary` `:1747` | 3 (`Dashboard.js`, `Loans.getLoanReports`, `jobs.updateMetrics`) | positional, see §3 |
| R3 | `GET /runreports/Aging%20Detail?R_officeId=1` | `getAgingReport` `:1731` | 3 (`Dashboard.getAgingReport`, `jobs.arrearsNotification`, `jobs.checkOverdueTenureLoans`) | positional, see §3 |

> **[A] `ActiveLoansSummary` is not a stock Fineract report.** The stock report is named "Active Loans - Summary" ([V] `fineract/.../0002_initial_data.xml:12565`). The LendLot instance must have a custom `stretchy_report` row with this name. **Its SQL must be extracted from production** (`SELECT report_sql FROM stretchy_report WHERE report_name='ActiveLoansSummary'`) before the facade can reproduce it.

## 3. Positional report contracts

veend-tenant reads report rows **by index**. The facade must reproduce the exact column order.

**ActiveLoansSummary** `data[0].row[i]` ([V] `controllers/Dashboard.js:1726-1753`, `:4766-4767`; `controllers/Loans.js:1712-1713`):

| idx | Meaning (per code comments) |
|---|---|
| 6 | principal repaid |
| 7 | principal outstanding |
| 8 | principal overdue |
| 9 | total interest |
| 10 | interest repaid |
| 11 | interest outstanding |
| 12 | interest overdue |
| 14 | fees repaid |
| 0–5, 13 | [A] unknown; capture from production |

**Aging Detail** `data[n].row[i]` ([V] `services/queue/jobs.js:11092-11108`, `:7938-7939`; `controllers/Dashboard.js:5472`):

| idx | Field |
|---|---|
| 0 | branch (office name) |
| 1 | currency |
| 2 | client account no |
| 3 | client name |
| 4 | loan account number (zero-padded; `loanId = row[4].replace(/^0+/,'')`) |
| 5 | loan amount |
| 6 | original principal |
| 7 | original interest |
| 8 | principal paid |
| 9 | interest paid |
| 10 | principal overdue |
| 11 | interest overdue |
| 12 | days in arrears |
| 13 | weeks-in-arrears band |
| 14 | days-in-arrears band (e.g. `"0 - 30"`) |

These rows are persisted into the global `Cbaloanarrears` collection by `jobs.checkOverdueTenureLoans` ([V] `jobs.js:11045-11110`).

## 4. `GET /loans/{id}` response fields consumed

Counts are reads across `controllers/` and `services/queue/` ([V], grep).

| Field | Reads | Used for |
|---|---|---|
| `feeChargesAtDisbursementCharged` | 72 | Net disbursement amount; treats the upfront fee as already repaid |
| `repaymentSchedule.totalRepaymentExpected` | 31 | Local `totalExpectedRepayment`; mandate amount |
| `repaymentSchedule.totalPrincipalDisbursed` | 19 | Local `amount`, `disbursementAmount` |
| `repaymentSchedule.totalOutstanding` | 14 | Local `totalOutstanding` |
| `repaymentSchedule.totalInterestCharged` | 12 | Local `interestAmount` |
| `interestRatePerPeriod` | 12 | Local `interestRate` |
| `repaymentSchedule.totalFeeChargesCharged` | 9 | Local `instalmentFee` |
| `status.waitingForDisbursal` / `pendingApproval` / `active` / `closed` / `overpaid` | 8/5/1/3/3 | State guards |
| `repaymentSchedule.totalRepayment` | 5 | Reconciliation |
| `summary.totalRepayment` / `summary.totalOutstanding` | 3/3 | Closure sync in Remita/IPPIS jobs |
| `repaymentSchedule.periods[]` | 3+ | Mandate schedule; on-lending schedule copy (`period`, `dueDate` as `[y,m,d]`, `totalDueForPeriod`, `totalOutstandingForPeriod`, `complete`) |
| `transactions[].{id, type.code, amount, manuallyReversed}` | 2 | `reverseRepayment` matches by **amount** |

> Date fields come back as Fineract arrays `[yyyy, m, d]`. Callers do `dueDate.join('-')` ([V] `lendlot/index.js:99,103`). The facade must keep this encoding.

## 5. Dead client methods (0 live callers; not part of the contract)

`getLoanRepayment`, `getClientAccount`, `getClientEmail`, `getClients`, `getEmployers`, `createEmployer`, `updateEmployer` (offices), `getAllStaff`, `createStaff`, `updateStaff`, `getLoanProductsCharges` (direct), `addChargeToLoanProduct`, `getChargeTemplate`, `getCharges`, `updateLoan`, `getAllLoans`, `deleteLoanCharge`.

[R] Remove these from the veend-tenant client during the migration (D6 step M1).

## 6. Endpoints Fineract has that veend-tenant never uses (but Fineract-Mod MVP needs)

| Capability | Fineract endpoint | Why it's needed |
|---|---|---|
| Write-off | `POST /loans/{id}/transactions?command=writeoff` | Local `writeOffLoan` must post to the GL (D2 defect DF-04) |
| Recovery payment | `?command=recoverypayment` | Collections after write-off |
| Waive interest/charges | `?command=waiveinterest`, `/charges/{id}?command=waive` | Settlements and discounts |
| Undo disbursal | `?command=undodisbursal` | Failed-transfer correction after disburse |
| Journal entries / trial balance | `/journalentries`, `/runreports/TrialBalance` | Finance reporting |
| Business date / COB | `/businessdate`, `/loans/catch-up` | Daily processing |
| Top-up | `isTopup` + `loanIdToClose` on create | Real top-ups (currently simulated locally) |
