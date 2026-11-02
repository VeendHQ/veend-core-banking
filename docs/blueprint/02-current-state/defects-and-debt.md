# D2.6 Verified Defects and Technical Debt

Severity: **S1** = financial misstatement or money-at-risk; **S2** = integrity or security; **S3** = maintainability.
"Fineract-Mod response" says how the new CBA neutralises the defect, and whether veend-tenant must change.

## 1. Financial correctness

| ID | Sev | Defect | Evidence | Fineract-Mod response |
|---|---|---|---|---|
| DF-01 | S1 | **Every CBA date is "today".** Repayments, approvals and disbursements are never value-dated, so payroll deductions collected days earlier are booked late. That distorts arrears, accrual and penalties | [V] `lendlot/index.js:726,1509,1563`; Remita deductions ≥2 days old (`jobs.js` `processRepaymentFromRemita`) | The facade accepts `transactionDate`. veend-tenant change **M2**: pass the deduction or payment date |
| DF-02 | S1 | **Double-posting on retry.** `payoff` builds the reference as `${eligibility}000${Date.now()}` when there is no `purchaseReference`. If `loan.save()` fails after the CBA post, the job retry posts again | [V] `jobs.js:3667-3669` | Mandatory `Idempotency-Key` / `externalId` uniqueness per tenant + channel (BR-TXN-01). veend-tenant change **M2**: deterministic key from the provider reference |
| DF-03 | S1 | **Float arithmetic on money** in the local mirror (`*1`, `+`, `-`, `.toFixed(2)`) | [V] `jobs.js:3695-3699`, `Users.js` `repaymentAmount` | All CBA amounts are `NUMERIC(19,6)` / decimal.js. veend-tenant should read balances from the CBA, not compute them (**M4**) |
| DF-04 | S1 | **Write-off is never posted to the CBA.** The GL overstates the portfolio and keeps accruing interest on written-off loans | [V] `jobs.writeOffLoan` `jobs.js:13548` | Write-off command with GL posting (writeoff-provisioning module). veend-tenant change **M3** |
| DF-05 | S1 | **Reversal matches by amount**, so with two equal repayments the wrong transaction can be undone. The guard runs after the CBA undo | [V] `Loans.js:4549-4610` | Return `transactionId` on repayment and store it. Reverse by id or external ref. veend-tenant change **M2** |
| DF-06 | S1 | **Failed rejection is silent.** `undoApproval`/`rejectLoan` resolve `{isError:true}`, the local loan is `closed` while the CBA loan stays approved, and the portfolio is overstated | [V] `lendlot/index.js:1597,1624`; `processDisbursementStatus` | Facade returns real errors. Reconciliation job flags orphan approved loans |
| DF-07 | S1 | **Top-up creates a second concurrent loan.** `isTopup/loanIdToClose` is commented out and `loanToClose` is never set, so the dormant net-off never runs | [V] `Users.js:2191-2196, 2272, 2373-2387` | Native top-up (close the old loan from the new disbursement in one atomic command). [A] Business confirmation needed |
| DF-08 | S1 | **The disbursement-edit body bug.** `editDisbursements` sends `this.httpBody` (stale) instead of the built `body`, so the approved amount and tranche are never applied | [V] `lendlot/index.js:1534-1557` | Not reproduced; facade validates the payload schema strictly |
| DF-09 | S2 | **Vida repayments are not posted** to the CBA | [V] `docs/vida/overview.md:308` | Parity backlog; veend-tenant change |
| DF-10 | S2 | **Upfront fee is netted from the transfer and treated as "repaid" locally** but booked as a disbursement charge in Fineract. Totals differ by the fee | [V] `Users.js:2254-2273` | CBA exposes `netDisbursementAmount` and `feesDeductedAtDisbursement`; veend-tenant reads them (M4) |
| DF-11 | S2 | **Disbursement date = requery date**, not the transfer date | [V] `processDisbursementStatus` → `disburseLoan` with today | Facade accepts `actualDisbursementDate`; veend-tenant passes the rail's value date (M2) |
| DF-12 | S2 | **No per-rail clearing accounts** (`paymentTypeId:1` always) | [V] `lendlot/index.js:728` | Payment types map to GL clearing accounts (accounting module) |

## 2. Integrity, security and operations

| ID | Sev | Defect | Evidence | Fineract-Mod response |
|---|---|---|---|---|
| DS-01 | S2 | **Credentials logged.** `APICalls` stores request headers, including Basic auth | [V] `services/request/index.js` (`Model.create(options)` with headers) | OAuth2 client credentials with short-lived tokens; header redaction in the logging middleware |
| DS-02 | S2 | **Default credentials fallback** in `config/production.js` (Fineract's well-known defaults) | [V] `config/production.js:52-55` | No defaults. Boot fails if the secret is missing (P0) |
| DS-03 | S2 | **TLS verification disabled** (`rejectUnauthorized:false`) | [V] `services/request/index.js` | Proper certificates. Client change M1 |
| DS-04 | S2 | **Hard-coded non-prod tenant** `veendhqautolending` | [V] `lendlot/index.js:45` | Per-tenant sandboxes in Fineract-Mod |
| DS-05 | S2 | **Shared mutable `this.httpBody`** leaks fields across calls and into GET query strings | [V] `lendlot/index.js` (e.g. `:64` passes `this.httpBody` on GET); `request` `options.qs = options.data` | New client (M1) is stateless |
| DS-06 | S2 | **Payload mismatch.** `LoansController.payoff` queues `{query, tenantDb, config}`, but `jobs.payoff` reads top-level `loanid/clientid/outstanding`. The query becomes `findOne({loanId: undefined})`; mongoose drops undefined keys, so this can match an arbitrary loan. It then fails only on the `clientid` check. `flutterwaveRepayment` and `paystackRepayment` enqueue `hook.data` **without `tenantDb/config`** (Cgate and CoralPay add them), so those payoffs fail unless the webhook data happens to contain them | [V] `controllers/Loans.js:465`; `jobs.js:4104, 4134` vs `4163-4166, 4197-4200`; `jobs.js:3666, 3690` | Typed request schemas on the facade; contract tests on the new client (M1/M2) |
| DS-07 | S2 | **Global `Cbaloanarrears` keyed by account number**, which collides across tenants | [V] `jobs.js:11111` | CBA provides tenant-scoped arrears via API |
| DS-08 | S3 | **Hard-coded GL ids, payment type, office and currency** | [V] `lendlot/index.js:840,870-881,728,497` | Product templates with role-based GL mapping; config per tenant |
| DS-09 | S3 | **`Object.prototype` monkey-patching** (`renameProperty`, `deleteExcept`) | [V] `lendlot/index.js:12-38` | Removed with the client rewrite (M1) |
| DS-10 | S3 | **Duplicate client.** `services/loandisk` is a fork of the LendLot client, and `search` is defined twice | [V] `loandisk/index.js` (28 `/api/v1` URLs); `lendlot/index.js:58,469` | Decommission LoanDisk mirroring [A business] |
| DS-11 | S3 | **The booking sequence is copy-pasted** in at least 6 jobs and 5 USSD handlers. God files (`jobs.js` 16.8k lines) | [V] `wc -l` | One `CbaClient.bookLoan()` composite operation on the facade (`POST /v1/loans/book`) |
| DS-12 | S3 | **About 22 dead client methods** | [V] [inventory §5](lendlot-endpoint-inventory.md#5-dead-client-methods-0-live-callers-not-part-of-the-contract) | Delete in M1 |
| DS-13 | S3 | **Custom report name `ActiveLoansSummary`** with an unknown definition | [V] not in stock Fineract | Extract the SQL from production; reimplement as a versioned report |

## 3. Root causes

1. **There is no CBA-side idempotency contract**, so every caller invents its own reference scheme.
2. **The CBA is treated as a calculator plus a ledger dump**, while business state (closure, default, write-off) lives in Mongo.
3. **The client was built for convenience, not correctness**: dates are implicit, errors are swallowed, and state is shared.

Fineract-Mod's design ([04-target-architecture](../04-target-architecture/overview.md)) addresses (1) and (3) at the platform level. Item (2) needs coordinated veend-tenant changes (D6 M2–M4).
