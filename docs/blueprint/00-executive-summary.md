# Executive Summary

## 1. What we currently have

**Apache Fineract (LendLot)** is a mature Java/Spring core banking system: about 7,100 source files, with strong loan, accounting, close-of-business and maker-checker capabilities ([D1.1](01-assessment/fineract.md)). We use a small part of it.

**veend-tenant** is our lending platform. It does eligibility, credit decisioning, collections and borrower experience itself. It uses Fineract mainly as a **loan calculator and ledger**, through one client class ([D1.2](01-assessment/veend-tenant.md)). Of 45 client methods, **23 are live**. They cover:
- clients
- loan products and charges
- loan calculate, create, approve, disburse and reject
- repayment and undo
- two custom-field tables
- two positional reports

The integration ([D2](02-current-state/architecture.md)) has **correctness problems that already affect the books today** (all verified in code; [D2.6](02-current-state/defects-and-debt.md)):
- **Every CBA posting is dated "today".** Payroll deductions collected days earlier are booked late (DF-01).
- **Retries can post the same repayment twice**, because many references are random or time-based (DF-02).
- **Write-offs never reach the CBA.** The GL overstates the portfolio and keeps accruing interest on written-off loans (DF-04).
- **Reversals match transactions by amount**, failed rejections are swallowed, top-ups create parallel loans, and Vida repayments are never posted (DF-05/06/07/09).
- Credentials are logged, TLS verification is off, and the production config falls back to default credentials (DS-01/02/03).

**bill-payment-portal (Giro)** is the second Fineract consumer, on its own Fineract tenant. It **owns the fixed-deposit (investor term-deposit) product**, internally "loan subscriptions" ([D1.4](01-assessment/bill-payment-portal.md), [D2.7](02-current-state/fixed-deposits-current-state.md)). These deposits are the **funding source of on-lending**: investor money flows to a pool, then to veend-tenant, then becomes a loan on a lender tenant. The current product opens a Fineract FD account but **never books deposits or interest in any ledger**. Interest is computed on read with float arithmetic, quarterly payouts fire only on an exact 90-day boundary, and tenure "2" means days in the API but months in the maturity maths (FD-01…FD-13).

**fineract-mod** is not a Fineract fork. It is an empty Node/TypeScript API framework (the same family as veend-tenant). It has none of the controls a ledger needs: no migrations, decimal money, transactions, locking, idempotency or authentication ([D1.3](01-assessment/fineract-mod-framework.md)).

## 2. What we need to build

**Fineract-Mod** is a **TypeScript modular monolith** on a hardened fineract-mod framework, with **Apache Fineract as the specification and test oracle** ([ADR-001](04-target-architecture/adr/ADR-001-typescript-rebuild.md)).

- **MVP scope** is lending parity, fixed deposits and a proper general ledger (16 modules, [D4](05-modules/README.md)):
  - platform core (tenancy, IAM, command bus, maker-checker, audit, business date)
  - organisation, clients and typed attributes
  - loan products, charges and the **schedule engine**
  - loan lifecycle and **loan transactions** (idempotent, value-dated, replayable)
  - **double-entry accounting**
  - COB (arrears, delinquency, penalties, accruals)
  - write-off, recovery and provisioning
  - reporting, events/webhooks, and a **LendLot-compatible API facade**
  - **fixed deposits**: products with explicit tenure units, multi-tranche funding, a locked rate, daily accrual, payout schedules, maturity and premature closure, and deployment records to on-lending, booked with Fineract's savings accounting roles ([spec](05-modules/fixed-deposits.md)). Giro keeps moving the money; the CBA computes and books it
- **Full microfinance-bank CBS** ([ADR-010](04-target-architecture/adr/ADR-010-full-mfb-core-banking-scope.md)). By stakeholder decision, Fineract-Mod must also provide **every feature in the recovery team's documents**: 67 features, tracked one by one in the [feature register](03-gap-analysis/recovery-team-feature-register.md). This adds 12 modules (about 31 in total):
  - savings, current and commitment accounts with overdrafts;
  - posting management (manual, batch, approvals, reversals);
  - teller, vault and branch operations;
  - payments, NIBSS, bills (Entrazact, PurePay) and mandates;
  - cards and disputes;
  - collateral, guarantors and mortgages;
  - restructuring and rescheduling;
  - recovery case management;
  - ECL impairment;
  - full CBN, NDIC and AML returns and credit-bureau submissions;
  - fixed assets and amortization, payroll, and shareholder management.

  For **BANK tenants** (e.g. Pettysave MFB, currently on BankOne), Fineract-Mod also moves money
- **Key design choices** ([D3](04-target-architecture/overview.md)):
  - PostgreSQL with schema-per-tenant isolation.
  - `NUMERIC` + decimal.js only.
  - One command = one ACID transaction (lock, post, audit, outbox).
  - Immutable balanced ledger with Fineract's posting-role vocabulary.
  - Server-side idempotency on every money movement.
  - OAuth2/OIDC with Fineract-compatible permission codes.
  - Transactional outbox → signed webhooks.
- **Retain, customise or build:** retain Fineract's *semantics* (status codes, transaction types, allocation strategies, posting rules, COB steps). Customise its *mechanisms* (tenancy, COB runner, events, datatables). Build everything as TypeScript code ([overview §4](04-target-architecture/overview.md#4-retain--customize--build)).

## 3. How we build it

A **dependency-ordered roadmap** of about 42 engineering weeks to the first lending-tenant cut-over, for a team of 4 backend engineers, 1 SDET, shared DevOps and a finance SME [A] ([D5](06-roadmap.md)). The full-MFB scope runs as parallel tracks for a **second squad**, reaching the BankOne (Pettysave) cut-over at about week 70 [A]. Without the second squad, add about 40 weeks:

| Phase | Outcome | Gate |
|---|---|---|
| P-D (parallel) | Production facts: Fineract version, products, CoA, custom reports, traffic samples, finance policies | Facts captured |
| P0 | Hardened framework + platform core | Security baseline |
| P1 | Organisation, clients, attributes, maker-checker | Compat clients green |
| P2 | Products, charges, **schedule engine** | **100% kobo-exact vs Fineract golden fixtures** |
| P3 | Ledger, loan lifecycle, transactions | Balanced E2E, golden allocation parity |
| P4 | COB, write-off, provisioning | 400-day time-travel parity |
| P5 | Reporting, events, compat facade, SDK | Recorded-traffic contract suite + pen test |
| P5-D (parallel) | Fixed deposits + Giro savings compat | Exact parity with Giro's current figures; sub-ledger = GL |
| P6 / P6-G | Migration and cut-over: lender tenants, and the Giro FD book (constructed, since it is off-ledger today) | Zero reconciliation diffs; finance-approved opening deposit liability |
| P6-A / B / C (squad 2) | Savings, current and commitment accounts; posting management; teller and branch; payments, NIBSS and bills | Golden CASA interest; teller EOD; NIBSS certification; 100% payment recon |
| P7-R / G / E / K (squad 2) | Collateral, restructuring, recovery; CBN, bureau and ECL; fixed assets, payroll, shareholders; cards | Pettysave's Sep-2026 CBN returns regenerated and reconciled; bureau files accepted; processor certification |
| P8-B | BankOne (Pettysave MFB) migration and cut-over | TB, BS, PAR, deposits and CBN returns reconcile |
| P9 | On-lending participation accounting, interest recalculation, groups, Kafka sink | — |

**Migration** ([D6](07-migration-strategy.md)) is tenant by tenant:
1. Full-history ETL that reuses Fineract ids, so veend-tenant references keep working.
2. Explicit, finance-approved correcting transactions for the known defects, applied to **both** systems.
3. A 14-day shadow run through the compat proxy with daily kobo-level reconciliation.
4. A cut-over of about 60 minutes per tenant.
5. A 30-day zero-data-loss rollback window.

veend-tenant needs **no code change to cut over** (phase M0). Small change packages (M1–M3) fix the defects above, and can be shipped **on LendLot now**.

## 4. Top risks ([D7](08-risk-register.md))

1. Semantic parity with Fineract (schedules, allocation, replay). Mitigation: golden fixtures, historical replay and parity gates.
2. The existing books are already inconsistent. Mitigation: explicit corrections with finance sign-off before the parallel run.
3. Credential hygiene and admin-surface bypass. Mitigation: rotate credentials, OAuth2, MCP disabled in production.
4. Team ramp-up on the banking domain. Mitigation: an embedded finance SME and this blueprint.
5. The investor deposit liability is unbooked today, and its balance-sheet owner is unclear. Mitigation: a constructed, exactly reconciled opening ledger, and a finance decision before the P5-D build.
6. Full-MFB scope effort and certification lead times (NIBSS, card schemes, PCI-DSS, CBN). Mitigation: a second squad, onboarding started in discovery, and build vs buy re-confirmed after BankOne discovery sizing (R-ARC-06, R-REG-03).

## 5. Decisions needed now

| # | Decision | Owner | Blocks |
|---|---|---|---|
| 1 | Approve production read access for discovery (P-D) | CTO | P2 |
| 2 | Accounting policy: upfront vs periodic accrual; write-off and provisioning policy | Finance | P3/P4 |
| 3 | Top-up behaviour: keep parallel loans or adopt atomic top-up | Product | P3 |
| 4 | Schedule veend-tenant M1–M3 (independently valuable on LendLot) | Engineering | P6 |
| 5 | Legal review: GPL-3.0 framework base; Apache NOTICE handling; external product name | Legal | External distribution |
| 6 | On-lending target model (participation vs sale vs inter-tenant loan) | Finance, legal | P9 |
| 7 | Fixed deposits: balance-sheet owner (treasury vs lender tenant), accrual start (funding vs deployment date), tenure "2" meaning, premature closure, WHT; and whether the Giro wallet ledger moves to the CBA | Finance, treasury, product | P5-D |
| 9 | Approve a second squad, compliance SME and NIBSS/card-processor onboarding; sponsor-bank vs direct NIBSS model; identify Entrazact and PurePay contracts | Leadership, operations | P6-A…P8-B |
| 8 | **Recovery-team input** ([review](09-input-reviews/recovery-team-2026-10.md)): (a) ✅ decided: all recovery-team features are in scope (ADR-010); (b) which entity is the unbalanced trial balance? (c) which classification basis is right for Pettysave's CBN 761: its own PAR report implies about ₦100m more provision; (d) is the guide input to configuring BankOne or to Fineract-Mod (build vs buy)? | Leadership, finance, compliance | Scope / ADR-001 |

## 6. Evidence and confidence

- Every current-state claim is tagged **[V]** with a file and line or a class reference into `fineract` or `veend-tenant`. Proposals are tagged **[R]**. Open facts that need production data or business input are tagged **[A]**, and the roadmap's discovery phase (P-D) exists to close them.
- The biggest open facts:
  - the production Fineract version
  - the SQL of the custom `ActiveLoansSummary` report
  - the allocation strategy and rounding configured on live products
  - whether LendLot runs COB or accrual jobs
