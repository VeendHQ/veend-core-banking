# D1.2 veend-tenant: Technical Assessment

Repository: `quantinum/veend-tenant`, HEAD `b5c70d40`. This is the lending management system that runs on Fineract (LendLot). It is the **only consumer** whose behaviour Fineract-Mod must preserve.

## 1. Technology stack

| Aspect | Finding | Evidence |
|---|---|---|
| Runtime | Node.js 18 (Docker `node:18-alpine`), CommonJS JavaScript | [V] `Dockerfile` |
| Web | Express 4.18, helmet 3, express-limiter, express-validator 6 | [V] `package.json` |
| Data | MongoDB via mongoose 6, **one database per tenant** (`Model.override(tenantDb)`). Sequelize is present but disabled | [V] `services/database/mongoOverride.js`; `services/database/index.js` (SQL commented out) |
| Queue | kue (a fork, `iolufemi/kue`) on Redis. Separate `clock` and `workers` processes | [V] `services/queue/{clock,workers,jobs}.js`, `clock/`, `workers/` Dockerfiles |
| HTTP client | `request-promise` (deprecated) wrapped by `services/request/index.js` with Mongo-logged replay cache | [V] |
| Auth | passport: Bearer (access tokens), JWT, plus "internal" and partner strategies | [V] `services/passport/index.js:62,149,220,251` |
| Money arithmetic | JS `Number` (floating point) everywhere, e.g. `loan.totalOutstanding * 1 - data.outstanding * 1` | [V] `services/queue/jobs.js:3695` |
| Monitoring | New Relic, Bugsnag, winston + Loggly | [V] `package.json` |
| Size | `jobs.js` 16,853 lines; `Webhooks.js` 6,861; `Dashboard.js` 5,917; `Users.js` 4,895; `Loans.js` 4,822 | [V] `wc -l` |

The codebase is built on an older JS version of the same API template that `fineract-mod` derives from. Both list the same author in `package.json`, and both have the same `routes/`, `services/queue`, `services/request` and `Trash`/`RequestLogs` patterns.

## 2. Structure

| Path | Role |
|---|---|
| `controllers/` | HTTP handlers. Fineract-heavy: `Users.js` (`acceptTerms`, `disburseLoan`), `Webhooks.js` (USSD, payment webhooks), `Loans.js` (payoff, `reverseRepayment`, on-lending), `Dashboard.js` (approve, reports, schedules), `Loanproducts.js`, `Lendlotcharges.js` |
| `models/` | About 40 mongoose schemas (see [data-model](../02-current-state/data-model.md)) |
| `routes/` | One router per controller, versioned (`*.v1.js`) |
| `services/lendlot/` | **The Fineract client** (1,759 lines), the single integration point |
| `services/loandisk/` | A fork of the LendLot client used for LoanDisk mirroring (28 `/api/v1` URLs) |
| `services/{remita,giro,ippis,coralPay,onepipe,vida,stanbic,mono,decision,creditcheck,identitypass,mybankstatement,termii,mail,google,hellosign}` | External integrations |
| `services/queue/jobs.js` | All background business logic: eligibility, disbursement, repayment, sync, on-lending, Vida, metrics |
| `config/` | Per-environment defaults. Per-tenant overrides live in `Accesstokens.config` |
| `scripts/` | Clock registrations and backfills |

## 3. Business capabilities (lending model)

| Capability | Implemented where | Local or CBA |
|---|---|---|
| Borrower onboarding, KYC (BVN/NIN), bank-statement analysis | `jobs.verifyBVN`, `retrieveBankStatement`, `services/identitypass`, `services/mybankstatement`, `services/mono` | Local |
| **Eligibility and offer**: salary × DTI via Remita/IPPIS payroll data, pre-approvals, decision engine | `jobs.getEligibility` (`jobs.js:360`, about 1,870 lines), `services/decision`, `services/creditcheck` | Local; repayment figure from CBA `calculateLoanSchedule` |
| **Loan application, approval and booking** | `Users.acceptTerms` (`controllers/Users.js:2081`), USSD handlers in `Webhooks.js`, `jobs.createLoan` (`:5889`) | CBA (create + approve) |
| Top-up | `Users.acceptTerms` gate (`isLoanRepaidByOutstandingThreshold` or Remita collection rate ≥ threshold) | **Local only**: Fineract top-up is commented out ([V] `Users.js:2191-2196`) |
| **Disbursement** via Giro or Remita | `jobs.disburseLoan` (`:2233`), `jobs.requeryDisbursement` (`:3579`), `jobs.processDisbursementStatus` (`:2996`) | Money moved externally; CBA `disburse` posted after confirmation |
| Mandate setup (Remita, OnePipe, Vida) | `jobs.getNewRemitaMandate`, `loanMandateSetupOnePipe`, `setupVidaMandate` | Local plus external |
| **Repayment collection** (payroll deduction, card, USSD, transfer, auto-debit) | `jobs.payoff` (`:3663`), `processRepaymentFromRemita` (`:4508`), `processRepaymentFromIppis` (`:4864`), `*Repayment` webhook jobs, `autoDebitBorrower` | Collected externally; CBA `repayment` posted |
| Repayment reversal | `Loans.reverseRepayment` (`controllers/Loans.js:4549`) | CBA `undo` + local |
| Arrears, defaults, recovery assignment | `sweepLoanDefaults`, `markLoanDefaulted`, `autoAssignOverdueLoansToRecoveryAgent`, `arrearsNotification` | **Local** (reads the CBA aging report) |
| Write-off | `jobs.writeOffLoan` (`:13548`) | **Local flag only**: never posted to the CBA |
| Insurance claim | `jobs.insuranceClaimed` (`:13588`) | Local only |
| Referral and recovery commissions | `payoff`, `acceptTerms` via Giro payment requests | Local plus Giro |
| On-lending / prime (cross-tenant funding) | `processOnLendingLoan` (`:8369`), `disburseOnlendingCyclicLoan`, `bookExternalPrimeLoan` (`:15672`) | CBA of the **funding** tenant |
| Vida BNPL | `logPendingVidaBnplLoansOnCba` (`:16127`), `reconcileVidaBnplBooking` (`:16681`) | CBA |
| Portfolio and aging dashboards | `Dashboard.portfolio`, `getAgingReport`, `jobs.updateMetrics` | CBA reports |
| LoanDisk mirroring (legacy) | `makeRepayment` → `Loandisk.makeRepayment` when `loan.loanDiskId` is set | External |

## 4. How it talks to Fineract (summary; detail in D2)

- [V] There is a single class, `services/lendlot/index.js`. Its constructor is `Lendlot(username, password, baseUrl, tenantId, reference)` (`:39`), and it is instantiated about 85 times across controllers and jobs.
- [V] Auth is HTTP **Basic** plus the `Fineract-Platform-TenantId` header (`getHeaders`, `:48-56`).
- [V] Outside production the tenant id is forced to `veendhqautolending` (`:45`).
- [V] All calls go through `services/request/index.js`. It logs every request and response to the Mongo `APICalls` collection. If the same `(RequestId, service)` previously returned 200, it **returns the cached response without calling Fineract**, which is the only idempotency mechanism.
- [V] All dates sent are "today" from `moment()` in the server's timezone: `transactionDate`, `approvedOnDate`, `actualDisbursementDate`, `submittedOnDate`. **Nothing is ever backdated**, including payroll deductions that happened days earlier (`makeRepayment` `:726`).
- [V] Of 45 client methods, **23 have live callers** and about 22 are dead. See [lendlot-endpoint-inventory](../02-current-state/lendlot-endpoint-inventory.md).

## 5. Business rules: local vs. delegated

| Delegated to Fineract | Implemented locally (and must stay in veend-tenant or move deliberately) |
|---|---|
| Interest and schedule computation (`calculateLoanSchedule`, loan creation) | Eligibility, credit decisioning, DTI, offers |
| Charges at disbursement and per installment (product charges) | Top-up gate. The net-off of the previous outstanding exists but is dormant, because `loanToClose` is never set |
| Repayment allocation (product's `transactionProcessingStrategyId`; the first option of the template is taken) | Arrears days, default flags, recovery workflow |
| Loan status (approved/active/closed/overpaid) | **Write-off** and **insurance claim** (local flags only) |
| Accounting (accrual-upfront, GL ids hard-coded) | Referral and recovery commissions |
| ActiveLoansSummary and Aging reports | Local mirror of balances (`Loans.totalOutstanding`, `totalRepayment`), updated with float arithmetic |

**Duplicated logic [V]:** Outstanding balance and closure are computed in both systems.
- Locally: `payoff` closes the loan when `totalExpectedRepayment - totalPaid < 1` (`jobs.js:3702`).
- In Fineract: `processRepaymentFromRemita` trusts Fineract's `status.closed` or `status.overpaid`.

The two can disagree. See [defects-and-debt](../02-current-state/defects-and-debt.md).

## 6. Tenancy model

- [V] One lender is one `Accesstokens` document (`models/Accesstokens.js`). It holds a `dbName` (that tenant's Mongo database) and `config.*`, including `lendlotUsername`, `lendlotPassword`, `lendlotBaseUrl` and `lendlotTenantId`. `services/passport` merges this config into `req.config`, and `clock.js` merges it into each job's `data.config`.
- [V] `jobs.createTenant` (`jobs.js:6872`) calls an external **tenant-creator** service (`config.tenantCreatorURL`) to provision a Fineract tenant. It then writes the shared `tenantUrl`, `tenantUsername` and password into the tenant config. [A] The tenant-creator service is not in any repo we inspected.
- [V] Offices are not used. `officeId: 1`, `branch: 1` and `R_officeId=1` are hard-coded throughout.

## 7. Strengths

- The CBA boundary is clean: everything goes through one client class, which makes the strangler swap tractable.
- There is a rich set of collection channels with existing webhook staging (`Webhooks` collection) and reconciliation jobs.
- Some flows already have documented "book at most once" rules (`docs/vida/overview.md`).

## 8. Weaknesses and technical debt (headline; full list in D2)

1. Financial arithmetic uses JS floats, and the local balance mirror can drift from the CBA.
2. Idempotency depends on caller-chosen `RequestId`s. Many are `Math.random()` or `Date.now()`, so retries can double-post.
3. Value dates are always "today", so backdated collections are booked at the wrong date. That distorts arrears and accrual.
4. Write-offs are not posted to the GL, so Fineract keeps accruing interest on written-off loans and the balance sheet overstates the portfolio.
5. Shared mutable `this.httpBody` leaks between calls, and there is a disbursement bug (`editDisbursements` sends the wrong body).
6. Credentials are logged in `APICalls` (the Basic auth header), and Fineract's default credentials are the fallback in production config (`config/production.js:52-55`).
7. Code is duplicated widely: the booking sequence appears in at least 6 jobs and 5 USSD handlers, and `loandisk` is a fork of the client.
8. The god files `jobs.js` and `Webhooks.js` make change risky.

## 9. Implications for Fineract-Mod

- **Compatibility contract:** the 23 live endpoints and the specific response fields veend-tenant reads (documented in D2) define the minimum LendLot-compat facade.
- **Fixing via the facade [R]:** Fineract-Mod can fix several defects at the boundary without touching veend-tenant. Examples: server-side idempotency keys, accepting an explicit `transactionDate`, and rejecting duplicate external references.
- **Moving rules into the CBA [R]:** write-off, default/NPA classification and top-up should become CBA operations. That way the GL reflects reality. These changes require small, explicit veend-tenant changes (see D6).
