# ADR-002: PostgreSQL, decimal money, Sequelize with umzug migrations

- Status: Proposed [R]

## Context

- The framework supports MySQL, Postgres, SQLite and MSSQL through Sequelize 6, with `.sync()` and no migrations, no DECIMAL and no transactions ([V] D1.3 G1–G4).
- Fineract supports MariaDB, MySQL and Postgres ([V]).
- A CBA needs exact arithmetic, ACID transactions with row locks, deferred constraints (balanced journals), JSONB, and schemas for tenancy.

## Decision

1. **PostgreSQL 16** is the only supported database. We need deferrable constraint triggers, `SKIP LOCKED`, schemas and roles per tenant, JSONB, and `NUMERIC`.
2. **Money** is stored as `NUMERIC(19,6)` and handled in code as `decimal.js`. Sequelize is configured so that DECIMAL returns strings (`dialectOptions: { decimalNumbers: false }` for pg is the default string behaviour). Repositories convert to `Decimal`, and ESLint bans `number` arithmetic on money types.
3. **ORM:** keep **Sequelize 6** to stay within the framework idiom, but:
   - use **umzug** for SQL-file migrations (platform and tenant runners);
   - use **managed transactions** with `cls-hooked`/AsyncLocalStorage namespace, so all queries in a command join the transaction automatically;
   - use `lock: Transaction.LOCK.UPDATE` and raw SQL for hot paths (posting, COB selection);
   - remove `.sync()` from all models (enforced by a lint rule).
4. Alternative considered: **Kysely** (typed SQL builder). It is better for complex SQL, but adds a second data-access idiom. It is revisit-able for the reporting module only.

## Consequences

- MySQL and MSSQL support from the template is dropped for Fineract-Mod.
- The test harness needs Testcontainers Postgres (the framework CI uses MySQL; [V] `.github/workflows/ci.yml`).
- Migration from LendLot (MariaDB/MySQL [A]) needs type mapping, e.g. `DECIMAL(19,6)` → `NUMERIC(19,6)` is lossless.
