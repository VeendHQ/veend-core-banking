# ADR-003: Schema-per-tenant on a shared PostgreSQL cluster

- Status: Proposed [R]

## Context

- Fineract uses **DB per tenant** with routing data sources ([V] `RoutingDataSource`, `JdbcTenantDetailsService`).
- veend-tenant maps one lender to one Fineract tenant ([V] `Accesstokens.config.lendlotTenantId`) and one Mongo database.
- Lenders are regulated institutions and expect data segregation. Per-tenant export and restore is a plausible contractual requirement [A].
- On-lending creates real cross-tenant financial relationships ([V] F5).

## Options

| Option | Isolation | Ops cost | Cross-tenant | Migration effort |
|---|---|---|---|---|
| DB per tenant (Fineract) | Strongest | High: N pools, N backups | Hard | N runs |
| **Schema per tenant** | Strong (schema + role) | Medium: one cluster, one pool, N schemas | Explicit platform service | N runs (parallel) |
| Shared tables + `tenant_id` + RLS | Policy-based | Lowest | Easy (too easy) | 1 run |

## Decision

**Schema per tenant**, with a `platform` schema for the registry and inter-tenant records:
- Each tenant has a dedicated DB role. Transactions run `SET LOCAL ROLE tenant_x; SET LOCAL search_path = t_x, platform`.
- Tenant resolution comes only from the authenticated principal.
- Cross-tenant flows (on-lending, participation) use a platform saga service. They never use cross-schema joins inside business commands.
- Escape hatch: a very large tenant can be moved to its own cluster by changing its registry DSN. The schema layout is identical.

## Consequences

- The migration runner must handle about 50–200 schemas [A tenant count] in parallel with per-tenant version tracking.
- Cross-tenant platform reporting (Veend-level BI) uses the read replica and `UNION ALL` views generated per tenant, or ETL to a warehouse.
- Connection pooling: one pool, with role and path switched per transaction. PgBouncer must run in **transaction** mode, and `SET LOCAL` is compatible with that.
