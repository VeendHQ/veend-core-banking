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

A **dependency-ordered roadmap** of about 42 engineering weeks to the first tenant cut-over, for a team of 4 backend engineers, 1 SDET, shared DevOps and a finance SME [A] ([D5](06-roadmap.md)):

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
| P7 | Rescheduling, on-lending participation, interest recalculation, later savings/deposits | — |

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

## 5. Decisions needed now

| # | Decision | Owner | Blocks |
|---|---|---|---|
| 1 | Approve production read access for discovery (P-D) | CTO | P2 |
| 2 | Accounting policy: upfront vs periodic accrual; write-off and provisioning policy | Finance | P3/P4 |
| 3 | Top-up behaviour: keep parallel loans or adopt atomic top-up | Product | P3 |
| 4 | Schedule veend-tenant M1–M3 (independently valuable on LendLot) | Engineering | P6 |
| 5 | Legal review: GPL-3.0 framework base; Apache NOTICE handling; external product name | Legal | External distribution |
| 6 | On-lending target model (participation vs sale vs inter-tenant loan) | Finance, legal | P7 |
| 7 | Fixed deposits: balance-sheet owner (treasury vs lender tenant), accrual start (funding vs deployment date), tenure "2" meaning, premature closure, WHT; and whether the Giro wallet ledger moves to the CBA | Finance, treasury, product | P5-D |

## 6. Evidence and confidence

- Every current-state claim is tagged **[V]** with a file and line or a class reference into `fineract` or `veend-tenant`. Proposals are tagged **[R]**. Open facts that need production data or business input are tagged **[A]**, and the roadmap's discovery phase (P-D) exists to close them.
- The biggest open facts:
  - the production Fineract version
  - the SQL of the custom `ActiveLoansSummary` report
  - the allocation strategy and rounding configured on live products
  - whether LendLot runs COB or accrual jobs
