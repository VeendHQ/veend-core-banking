# D1.3 fineract-mod Framework: Assessment

Repository: `quantinum/fineract-mod`, HEAD `61fdfc3`. **This repo is not a Fineract fork.** It is a clean copy of the "Express REST API and MCP Server Framework" (Node 22+, TypeScript, Express 4, Mongoose 8, Sequelize 6, Bull, MCP SDK; [V] `package.json`). It has **no banking domain code**. Fineract-Mod will be built on top of it ([ADR-001](../04-target-architecture/adr/ADR-001-typescript-rebuild.md)), so this assessment lists what we reuse and what we must harden.

## 1. What the framework provides

| Component | Behaviour | Evidence |
|---|---|---|
| Bootstrap | Cluster fork per CPU in production. `initializeDatabases()` needs Mongo **and** log Mongo; Redis is needed in production; SQL is optional (a failure only warns) | [V] `src/app.ts`, `src/services/database/init.ts` |
| Middleware chain | helmet, cors, body parsers (50 MB), request id (fnv), async request logging via Bull, response helpers, encryption interpreter, hpp, content-length, Redis rate-limit, Redis GET cache (90 s), routes, 404, error handler | [V] `src/routes/index.ts` |
| Route loader | Auto-imports `src/routes/*`. `x.v1.ts` mounts at `/v1`, and the latest version is also mounted at `/`. The "latest" lookup assumes contiguous versions | [V] `src/routes/index.ts` |
| Models | Auto-loaded. Mongo models on `db.mongo`; SQL models via `db.sql.define` + **`.sync()` at import** | [V] `src/models/index.ts`, `template/model_sql.tmpl` |
| Code generator | `npm run generate -- --name X [--sql]` → route, model, controller, tests, MCP registration | [V] `gulpfile.cjs` (`service` task), `template/*.tmpl` |
| Responses | `res.ok/badRequest/forbidden/notFound/serverError/unauthorized/unprocessable` → `{status, data, requestId}` | [V] `src/services/response/*` |
| Validation | Only "required, non-empty" for `req._required` fields; zod is installed but not used for requests | [V] `src/services/validator/index.ts` |
| "Encryption" | `GET /initialize` issues a PBKDF2 "x-tag". POST bodies marked `secure` are AES-CTR encrypted with a predictable counter and a SHA-512 "truth" hash | [V] `src/services/encryption/index.ts` |
| Queue | Bull: `workers.ts` (searchIndex, logRequest, saveToTrash, sendWebhook, sendHTTPRequest) and `clock.ts` (cron rows from a Mongo `Clock` model) | [V] `src/services/queue/*` |
| Trash | On delete, the record is copied to Mongo `Trash` via the queue, then **hard-deleted** (`paranoid` commented out) | [V] `template/controller_sql.tmpl`, `template/model_sql.tmpl` |
| MCP | Unified MCP server at `/mcp/http`; models are exposed as CRUD tools (`create_<model>` → `Model.create(args)`) | [V] `src/mcp/servers/*`, `src/mcp/auto-register.ts` |
| Testing | mocha + chai + sinon + supertest + nock, c8 coverage. Needs live Mongo, Redis and MySQL | [V] `test/run.mjs`, `.mocharc.json`, `.github/workflows/ci.yml` |
| Deploy | Multi-stage `node:24-alpine` Dockerfile with `/health/live`. docker-compose with MySQL 8, Mongo and Redis; separate clock and worker images | [V] `Dockerfile`, `docker-compose.yml`, `clock/`, `workers/` |

## 2. Gaps for a core banking system (all verified in code)

| # | Gap | Why it matters for a CBA | Evidence |
|---|---|---|---|
| G1 | **No schema migrations**, only `Model.sync()` | A ledger schema must be versioned, reviewable and reversible | [V] no umzug/sequelize-cli/migrations dir |
| G2 | **No decimal money type** (no `DECIMAL` anywhere) | Float money causes kobo drift | [V] grep `DECIMAL` in `src`, `template` = 0 |
| G3 | **No DB transactions** (`Model.transaction` is an unbound, unused reference) | Postings and balance updates must be atomic | [V] `template/model_sql.tmpl`; `// ToDo: Test transactions` in `controller_sql.tmpl` |
| G4 | **No row locking or optimistic concurrency** | Two concurrent repayments on one loan would corrupt balances | [V] no `lock:`/`FOR UPDATE`/`version` |
| G5 | **No idempotency keys** | Payment webhooks and retries must be exactly-once | [V] |
| G6 | **No authentication, RBAC or tenancy**. `req.accountId/appId/developer` are read but never set | Fundamental | [V] `src/routes/index.ts`, `src/config/index.ts` |
| G7 | **Bulk PUT/DELETE pass `req.query` straight into `where`**, so an empty query hits every row | Mass-mutation hazard | [V] `template/controller_sql.tmpl` |
| G8 | **MCP exposes raw model CRUD, unauthenticated**, bypassing HTTP middleware | An LLM tool could write to the ledger | [V] `src/mcp/servers/models/tools.ts` |
| G9 | Audit is async via the queue, not tamper-evident; response logging is commented out | Regulators expect a complete, immutable audit | [V] `src/services/response/ok.ts` |
| G10 | GET cache keyed on url+ip+UA for 90 s | Stale balances served after a repayment | [V] `src/services/cache/redis-cache.ts` |
| G11 | Home-grown payload "encryption" (predictable CTR, hash not MAC) | False sense of security; use TLS plus real auth | [V] `src/services/encryption/index.ts` |
| G12 | `searchIndex` copies all string fields into a plain-text `tags` column | PII duplication (BVN, phone) | [V] `template/model_sql.tmpl` hooks |
| G13 | Mongo is mandatory even for SQL apps | Two datastores to operate; Trash, Logs and Clock are Mongo-only | [V] `src/services/database/init.ts` |
| G14 | SQL pool is fixed at max 5; no read replicas | Throughput for COB | [V] `src/services/database/sql.ts` |
| G15 | Route "latest" detection breaks with non-contiguous versions | API versioning | [V] `src/routes/index.ts` |

## 3. Reuse decision

| Keep as is | Keep with hardening | Replace or disable for financial modules |
|---|---|---|
| Express app skeleton, helmet/cors/hpp, response helpers, request id | Route loader (fix versioning, G15); Bull queue + clock/workers (job registry in Postgres, not Mongo) | `Model.sync()`, replaced by versioned migrations (G1) |
| Logger (winston) | Rate limiting (key on authenticated principal + tenant) | Generated CRUD controllers, which **must not** be used for financial entities (G7) |
| Docker multi-stage build, health endpoints | Test harness (add Testcontainers Postgres, property tests) | GET cache, disabled on financial reads (G10) |
| TypeScript strict config, ESLint | MCP: **read-only**, authenticated, admin tenants only; off in production by default (G8) | Payload "encryption", replaced by TLS + OAuth2 (G11) |
| | Request logging, moved to a synchronous, append-only audit table for commands (G9) | Mongo-mandatory bootstrap, made optional (G13); Trash, replaced by soft-delete/immutable history in Postgres |

The concrete hardening tasks are P0 in the [roadmap](../06-roadmap.md#phase-p0--foundations).
