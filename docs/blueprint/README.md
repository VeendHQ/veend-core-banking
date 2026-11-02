# Fineract-Mod Blueprint

This is the research, architecture and delivery blueprint for **Fineract-Mod**, Veend's in-house Core Banking Application (CBA). Fineract-Mod replaces the Apache Fineract instance (internally called **LendLot**) that `veend-tenant` uses today.

> Status: Discovery and documentation phase. No production code has been changed in any repository.
> Prepared: 2026-11-02. Sources: `fineract` (Apache Fineract `8e80aa6e85`), `veend-tenant` (`b5c70d40`), `bill-payment-portal` / Giro (`25d8b550`, fixed-deposit product), `fineract-mod` (`61fdfc3`).

## The three questions this blueprint answers

| Question | Where |
|---|---|
| **What do we currently have?** | [01-assessment](01-assessment/), [02-current-state](02-current-state/) |
| **What do we need to build?** | [03-gap-analysis](03-gap-analysis/capability-matrix.md), [04-target-architecture](04-target-architecture/), [05-modules](05-modules/) |
| **How do we build it?** | [06-roadmap](06-roadmap.md), [07-migration-strategy](07-migration-strategy.md), [08-risk-register](08-risk-register.md) |

Start with the [Executive Summary](00-executive-summary.md).

## Document map

| # | Deliverable | Documents |
|---|---|---|
| D1 | Technical Assessment Report | [01-assessment/fineract.md](01-assessment/fineract.md), [01-assessment/veend-tenant.md](01-assessment/veend-tenant.md), [01-assessment/bill-payment-portal.md](01-assessment/bill-payment-portal.md) (Giro, fixed deposits), [01-assessment/fineract-mod-framework.md](01-assessment/fineract-mod-framework.md) |
| D2 | Current-State Architecture | [02-current-state/architecture.md](02-current-state/architecture.md), [lendlot-endpoint-inventory.md](02-current-state/lendlot-endpoint-inventory.md), [business-flows.md](02-current-state/business-flows.md), [data-model.md](02-current-state/data-model.md), [money-lifecycle.md](02-current-state/money-lifecycle.md), [defects-and-debt.md](02-current-state/defects-and-debt.md), [fixed-deposits-current-state.md](02-current-state/fixed-deposits-current-state.md) |
| — | Gap Analysis | [03-gap-analysis/capability-matrix.md](03-gap-analysis/capability-matrix.md) |
| D3 | Target Architecture | [04-target-architecture/](04-target-architecture/) (overview, data, API, security & tenancy, processing, operations, ADRs) |
| D4 | Module-by-Module Specs | [05-modules/](05-modules/) (one file per module, fixed 14-section template) |
| D5 | Development Roadmap | [06-roadmap.md](06-roadmap.md) |
| D6 | Migration & Integration Strategy | [07-migration-strategy.md](07-migration-strategy.md) |
| D7 | Technical Risk Assessment | [08-risk-register.md](08-risk-register.md) |

## Key decisions already taken

1. **Build approach.** Fineract-Mod is a **TypeScript rebuild** on the `fineract-mod` Node framework, hardened for banking. Apache Fineract serves as the **reference specification and test oracle**; we do not fork the Java code. See [ADR-001](04-target-architecture/adr/ADR-001-typescript-rebuild.md).
2. **MVP scope.** The first release covers **lending parity plus the general ledger**: clients, products, charges, schedules, the loan lifecycle, repayments and reversals, double-entry accounting, close-of-business (COB) and accruals, write-off and provisioning, RBAC, maker-checker, audit, multi-tenancy and reports. It also covers **fixed deposits**, the investor term-deposit product currently run by `bill-payment-portal` (Giro); see [05-modules/fixed-deposits.md](05-modules/fixed-deposits.md). Savings and wallet accounts, recurring deposits, groups, rescheduling and full participation accounting are documented as future phases.
3. **Migration style.** We use a **strangler migration**. veend-tenant switches CBA through a client interface and a LendLot-compatible API facade, one tenant at a time, with Fineract kept running in parallel until reconciliation passes. Giro's fixed deposits follow their own track ([D6 §12](07-migration-strategy.md#12-giro-fixed-deposit-migration)).

## Evidence convention

Every material claim carries one of these tags:

| Tag | Meaning |
|---|---|
| **[V]** | **Verified** in source code. It is followed by a reference such as `veend-tenant/services/lendlot/index.js:717`, or by a class or method name. Paths are relative to the sibling repos under `quantinum/`. |
| **[R]** | **Recommendation**: the architect's proposal. |
| **[A]** | **Assumption** or open question. It needs confirmation from the business, operations or production data before we rely on it. |

Line numbers refer to the commits listed at the top and will drift. Class and method names are the stable anchor.

## Conventions

- Diagrams are written in Mermaid and render on GitHub, GitLab and in VS Code with a Mermaid extension.
- Money is written in NGN, and "kobo-exact" means agreement to 0.01 NGN.
- No secrets are reproduced. Where configuration contains credentials, the docs give only the key name.
- Fineract enum ids are quoted where veend-tenant hard-codes them (for example `accountingRule: 4` = `ACCRUAL_UPFRONT`).

## Glossary

| Term | Meaning |
|---|---|
| **LendLot** | Veend's Fineract deployment; also the client class `veend-tenant/services/lendlot/index.js` |
| **CBA** | Core Banking Application, the system of record for loans and the general ledger |
| **Tenant / lender** | A Veend customer institution. In veend-tenant it is one `Accesstokens` document with its own Mongo database and its own Fineract tenant |
| **Client / borrower** | The end customer (a Fineract `m_client`; `Users` in veend-tenant) |
| **COB** | Close of Business, the end-of-day batch (accruals, arrears, delinquency, penalties) |
| **Business date** | The logical banking date. It can differ from the wall-clock date |
| **Maker-checker** | Four-eyes control: one user initiates a command and another approves it |
| **On-lending / prime** | Funding flow where one tenant's loan is financed by another tenant's book |
| **Vida BNPL** | Buy-now-pay-later product line booked onto the CBA |
| **Remita / IPPIS** | Nigerian payroll-deduction collection channels |
| **Giro** | Veend's wallet and transfer service, used for disbursement and collection. Its code is the `bill-payment-portal` repo, which also runs the fixed-deposit product |
| **Fixed deposit / loan subscription** | Giro's investor term deposit (`Accounts.isOnlending`). Tenure 2/3/6/12, quarterly or at-maturity interest. Funds on-lending |
| **Primelog** | Giro's Mongo record of one FD funding tranche and its deployment to on-lending |
| **Eden** | The synthetic borrower on the lending-to-lender tenant that receives FD-funded on-lending loans |
| **Datatable** | Fineract's extension tables (`userdata`, `loandata`) holding custom fields |
| **Golden fixture** | A recorded Fineract input/output pair used to check that Fineract-Mod calculates identically |
