# D2.1 Current-State Architecture

How veend-tenant uses Apache Fineract (LendLot) today. Every element below was verified in code unless tagged otherwise.

## 1. System context

```mermaid
flowchart LR
    subgraph Channels
        APP[Mobile / Web app]
        USSD[USSD aggregators<br/>Cgate, marketplace]
        DASH[Lender dashboard]
        PARTNER[Partners: Vida, external prime]
    end

    subgraph VT[veend-tenant - Node/Express]
        API[Express API<br/>controllers/*]
        WK[kue workers<br/>services/queue/jobs.js]
        CLK[clock<br/>services/queue/clock.js]
        LL[[services/lendlot<br/>Fineract client]]
        LD[[services/loandisk]]
        REQ[[services/request<br/>APICalls replay cache]]
    end

    subgraph Data
        MT[(Mongo: one DB per tenant<br/>Users, Loans, Repayments...)]
        MG[(Mongo global / log DB<br/>Accesstokens, APICalls, Cbaloanarrears)]
        RD[(Redis<br/>kue)]
    end

    subgraph CBA[LendLot = Apache Fineract]
        FAPI[fineract-provider /api/v1]
        FDB[(Fineract tenant DBs<br/>one per lender)]
        TC[tenant-creator service<br/>external, A]
    end

    subgraph Money[Payment and collection rails]
        GIRO[Giro wallet]
        REMITA[Remita]
        IPPIS[IPPIS]
        PSF[Paystack / Flutterwave / CoralPay]
        OP[OnePipe]
    end

    subgraph Decisioning
        DEC[Mono / Indicina / credit bureaus / BVN]
    end

    APP & USSD & DASH & PARTNER --> API
    API --> WK
    CLK --> RD --> WK
    API & WK --> MT & MG
    API & WK --> LL --> REQ --> FAPI --> FDB
    REQ -. logs every call .-> MG
    WK --> LD
    WK -- createTenant --> TC --> FDB
    API & WK <--> GIRO & REMITA & IPPIS & PSF & OP
    WK --> DEC
```

Key facts:
- [V] All CBA traffic goes through **one class**, `services/lendlot/index.js`, which in turn uses `services/request`.
- [V] **There are two systems of record for balances.** Fineract holds the authoritative schedule and transactions. veend-tenant `Loans` keeps a float-arithmetic **mirror** (`totalOutstanding`, `totalRepayment`, `status`) that drives user-facing behaviour.
- [V] Money movement (disbursement and collection) happens **outside** the CBA, through Giro, Remita, IPPIS, card processors and OnePipe. The CBA is told afterwards.

## 2. Tenant → Fineract mapping

```mermaid
flowchart TB
    subgraph Global Mongo
        AT1[Accesstokens: Lender A<br/>dbName=lenderA<br/>config.lendlotTenantId=lenderA<br/>config.lendlotBaseUrl=...]
        AT2[Accesstokens: Lender B ...]
    end
    subgraph Tenant Mongo DBs
        DA[(lenderA)]
        DB[(lenderB)]
    end
    subgraph Fineract
        TA[(tenant lenderA DB)]
        TB[(tenant lenderB DB)]
    end
    AT1 --> DA
    AT1 -- Basic auth + TenantId header --> TA
    AT2 --> DB
    AT2 --> TB
    DA -. on-lending: loan booked on funding tenant .-> TB
```

- [V] `services/passport` merges `Accesstokens.config` into `req.config`, and `clock.js` merges it into each job's `data.config`. Each `new Lendlot(...)` gets that tenant's credentials.
- [V] Outside production, every tenant is routed to the Fineract tenant `veendhqautolending` (`lendlot/index.js:45`), so staging cannot test multi-tenant isolation.
- [V] There is no office hierarchy: `officeId=1` everywhere.
- [V] **Cross-tenant on-lending:** `processOnLendingLoan` (`jobs.js:8369`) picks a funding lender by `Accesstokens.lastBalance` / `lastPrimeDisbursementAccountBalance` and creates a hidden "onlending" product on *that* lender's tenant. It then books client and loan there and links them through `Loans.onlenderLoanId` and `onlenderAccessToken`.

## 3. Integration architecture (CBA boundary)

```mermaid
flowchart LR
    subgraph Callers
        U[Users.acceptTerms]
        W[Webhooks USSD handlers]
        J[jobs: getEligibility, createLoan,<br/>disburseLoan, processDisbursementStatus,<br/>payoff, processRepaymentFromRemita/Ippis,<br/>on-lending, Vida, prime, sync, metrics]
        LC[Loans.reverseRepayment, payoff endpoints]
        D[Dashboard.portfolio / actions / aging]
        P[Loanproducts / Lendlotcharges CRUD]
    end
    subgraph Client[services/lendlot]
        M1[clients: search, createBorrower,<br/>getBorrower, updateBorrower]
        M2[products: getLoanProducts, createLoanProduct,<br/>updateLoanProduct, charges CRUD]
        M3[loans: template, calculateLoan, createLoan,<br/>approve, disburse, undoApproval, reject,<br/>getLoan, getLoanSchedule, datatable]
        M4[transactions: makeRepayment, undoRepayment]
        M5[reports: ActiveLoansSummary, Aging Detail]
    end
    U & W & J --> M1 & M3
    J & LC --> M4
    D --> M3 & M5
    J --> M5
    P --> M2
    M1 & M2 & M3 & M4 & M5 --> F[(Fineract /api/v1)]
```

The [endpoint inventory](lendlot-endpoint-inventory.md) is the authoritative list: 23 live methods covering about 25 distinct endpoint shapes.

## 4. Runtime topology

| Process | Entry | Fineract usage |
|---|---|---|
| API (`npm start`) | `app.js` | Synchronous calls inside request handlers. `acceptTerms` blocks on create → approve → get |
| Workers (`npm run workers`) | `services/queue/workers.js` | Most CBA writes: disbursement confirmation, all repayments, on-lending, Vida |
| Clock (`npm run clock`) | `services/queue/clock.js` | Enqueues cron jobs from DB rows (`requeryDisbursement`, `pushRepayments`, `*Repayment` hourly sweeps, `checkOverdueTenureLoans`, `updateMetrics`, …) |
| LendLot | Fineract `fineract-provider` behind `cba.veendhq.com:8443` ([V] `veendTenant*.yml` `TENANT_URL`) | System of record for loans and GL |

[A] Fineract's version, deployment topology, DB engine, COB/job schedule and maker-checker settings in production are **not** visible in these repos. They must be captured from the running LendLot instance before migration planning is final. See [D6 §2](../07-migration-strategy.md#2-pre-migration-discovery-on-the-live-lendlot).
