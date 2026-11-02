# ADR-001: Rebuild Fineract-Mod in TypeScript; use Apache Fineract as the specification

- Status: **Accepted** (stakeholder decision, 2026-11-02)
- Deciders: Engineering leadership

## Context

- `fineract-mod` is a Node/TypeScript framework template, not a Fineract fork ([V] D1.3).
- Apache Fineract is about 7,100 Java files on Java 25 / Spring Boot 4.1 ([V] D1.1). veend-tenant uses about 23 client methods covering roughly 25 endpoint shapes ([V] D2.2).
- Veend's engineering team works in Node.js. veend-tenant and the framework share lineage.

## Options

| Option | Pros | Cons |
|---|---|---|
| A. Fork Fineract (Java), trim and customise | The most proven financial logic on day one | Java/Spring skills gap; we would own about 7k files of which roughly 10% are used; upstream merge burden; DB-per-tenant ops |
| B. Hybrid: TS facade over a running Fineract, replaced module by module | Lowest initial risk | Two systems to operate long-term; the facade inherits Fineract's limits; the ledger stays in Fineract until the end |
| **C. TS rebuild, Fineract as spec and test oracle** | Team fit; scope limited to what we use plus the correctness gaps; full ownership | Must re-derive financial semantics correctly. Mitigated by ADR-009 golden tests and a parallel run |

## Decision

Option **C**. Fineract-Mod is a TypeScript modular monolith on the fineract-mod framework, hardened per ADR-008. Fineract's semantics (status codes, transaction types, allocation strategies, accounting roles, posting rules, COB steps) are the **reference behaviour**. Fineract instances are used to generate golden fixtures and to run in parallel during migration.

## Amendment (2026-11-02)

The scope was extended to a full microfinance-bank CBS by [ADR-010](ADR-010-full-mfb-core-banking-scope.md). The build decision stands, and should be re-confirmed after the BankOne discovery sizing (R-ARC-06).

## Consequences

- We must build a schedule engine and an allocation engine with kobo-exact parity. This is the highest-risk area (R-FIN-01). It is front-loaded in P2 with golden testing.
- We take on full ownership of correctness, so we need finance-domain review capacity (R-OPS-02).
- No Apache code is copied by default. If any is, NOTICE obligations apply (D1.1 §12).
- Fineract features outside the MVP (savings, deposits, groups, investor) are rebuilt later only if the business needs them.
