# ADR-008: Harden the fineract-mod framework for financial workloads

- Status: Proposed [R]

## Context

[V] D1.3 lists gaps G1–G15 in the framework (no migrations, no decimal, no transactions or locking, no auth, unsafe bulk endpoints, unauthenticated MCP CRUD, stale caching, mandatory Mongo, and others).

## Decision

| Gap | Change |
|---|---|
| G1 | umzug migrations; ban `.sync()` |
| G2 | `NUMERIC` + decimal.js; lint rule against money as `number` |
| G3/G4 | AsyncLocalStorage-scoped managed transactions; `FOR UPDATE` helpers; lock and statement timeouts |
| G5 | Idempotency middleware and store (ADR-005) |
| G6 | IAM middleware: OAuth2/OIDC, RBAC, tenant resolution |
| G7 | Financial modules do **not** use generated CRUD controllers. The generator stays available for non-financial reference data, with bulk PUT/DELETE removed from the templates |
| G8 | MCP disabled in production by default. When enabled: authenticated, read-only resources, admin tenants only. Any tool that writes must call the `CommandBus` |
| G9 | Synchronous `command_log` audit; request logs stay asynchronous (non-authoritative) |
| G10 | GET cache disabled for `/v1` and compat routes (`Cache-Control: no-store`) |
| G11 | Remove the x-tag "encryption" path from the Fineract-Mod routes; TLS + OAuth2 instead |
| G12 | Remove the `searchIndex` PII tagging; use explicit indexed search columns |
| G13 | Make Mongo optional; move Trash, RequestLogs and Clock to Postgres |
| G14 | Configurable pool size; read-replica connection for reports |
| G15 | Fix the route-version loader to pick the max version, not a contiguous count |

## Consequences

- These changes affect the shared template. [R] Do them in fineract-mod only, and offer them upstream as optional features. Do not change veend-tenant.
- P0 roadmap items map 1:1 to these rows.
