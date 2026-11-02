# D3.4 Authentication, Authorization and Multi-Tenancy

> [R] throughout. Decisions: [ADR-003](adr/ADR-003-tenancy-model.md), [ADR-005](adr/ADR-005-command-pipeline.md).

## 1. Identity

| Principal type | Mechanism | Used by |
|---|---|---|
| **Service client** | OAuth2 client-credentials. Short-lived JWT (10 min), `aud=fineract-mod`, claims `tenant`, `client_id`, `scopes`. mTLS optional for production | veend-tenant workers and API (one client per tenant, or one platform client with an explicit `tenant` claim limited to an allowlist) |
| **Human user** | OIDC (authorization code + PKCE) against the corporate IdP or the built-in user store. MFA (TOTP) mandatory for checker, finance and admin roles | Back-office UI |
| **Legacy Basic** | Username/password mapped to a service client, compat routes only, IP allowlist | veend-tenant before M1 |

Token validation is a single Express middleware. It sets `req.principal = {tenantKey, actorType, actorId, permissions[]}`. This replaces the framework's unset `req.accountId/appId/developer` (G6).

## 2. Authorization: RBAC with Fineract-style permission codes

- **Permission code** = `ACTION_ENTITY`, matching Fineract (`CREATE_LOAN`, `APPROVE_LOAN`, `DISBURSE_LOAN`, `REPAYMENT_LOAN`, `ADJUST_LOAN` (reversal), `WRITEOFF_LOAN`, `CREATE_JOURNALENTRY`, …). This allows migrated `m_role` and `m_permission` data to map directly.
- **Checker permission** = `<code>_CHECKER` (Fineract convention).
- **Roles** are tenant-scoped bundles of permissions. Users and service clients hold roles.
- **Data scope:** users may be restricted to an office subtree, and queries are filtered by `office.hierarchy LIKE '<path>%'`.
- **Enforcement:** each command and query declares its permission. The command bus checks it before the handler runs, so there is no way to bypass it through an alternate route (MCP included; see ADR-008).

### Baseline roles

| Role | Key permissions |
|---|---|
| `SERVICE_LENDING` (veend-tenant) | client CRUD, `CREATE/APPROVE/DISBURSE/REJECT/UNDOAPPROVAL_LOAN`, `REPAYMENT_LOAN`, `ADJUST_LOAN` (only within 24 h and below a threshold; above that it goes to maker-checker), `READ_*`, `RUN_REPORT_*` |
| `OPS_OFFICER` | Read all, initiate reversals, write-offs and waivers (maker) |
| `OPS_SUPERVISOR` | Checker for the above |
| `FINANCE` | GL accounts, manual JEs (maker), closures, provisioning runs, financial reports |
| `FINANCE_CONTROLLER` | Checker for manual JEs, closures and provisioning |
| `AUDITOR` | Read-only everything, including the audit log |
| `TENANT_ADMIN` | Users, roles, products, charges, config |
| `PLATFORM_ADMIN` | Tenant provisioning (platform schema only; no access to tenant business data by default) |

## 3. Maker-checker

- Configured per permission code in `maker_checker_config(permission_code, enabled, threshold_amount NULL)`, with optional amount thresholds. For example, an `ADJUST_LOAN` reversal needs approval when it is older than 1 day or larger than ₦500,000 [A: business to set].
- A held command is stored in `command_log` as `status = PENDING_APPROVAL`, together with the full validated payload and `maker_id`.
- **The checker must differ from the maker** (enforced). Approval re-validates against the **current** state, because state may have changed since submission, and then executes.
- Pending commands expire after a configurable TTL (default 72 h).
- Service clients can be given `checker superuser` only for specific automated commands. This mirrors Fineract's `isCheckerSuperUser` ([V] `PortfolioCommandSourceWritePlatformServiceImpl.java:80`).

## 4. Multi-tenancy

| Aspect | Design |
|---|---|
| Model | **Schema per tenant** in one Postgres cluster (ADR-003). The `platform.tenant` registry holds `key`, `schema`, `status`, `timezone`, `base_currency`, `config` |
| Resolution | From the authenticated principal's `tenant` claim **only**. The compat header `Fineract-Platform-TenantId` must equal it, otherwise `403` |
| Isolation | `SET LOCAL search_path` per transaction. The app DB role has no `USAGE` on other schemas in a tenant transaction. Implemented by switching the role: `SET LOCAL ROLE tenant_<key>`, each tenant role owning only its schema. Defence in depth: integration tests assert cross-tenant reads fail |
| Provisioning | `POST /platform/tenants` (PLATFORM_ADMIN) creates the schema, role, migrations, seed CoA template, default roles and admin user. Idempotent. This replaces the external tenant-creator service ([V] `veend-tenant` `jobs.createTenant`) |
| Per-tenant config | Timezone (business date roll), currency, rounding, COB schedule, webhook endpoints, maker-checker rules |
| Cross-tenant operations | Prohibited in normal commands. On-lending and participation is a **platform-level inter-tenant service** that runs two tenant transactions coordinated by a saga with compensations, recorded in `platform.inter_tenant_transfer`. See [future/investor](../05-modules/future/investor.md) |
| Non-production | Each tenant has a sandbox tenant (`<key>_sbx`). This removes the shared `veendhqautolending` pattern ([V] DS-04) |
| Treasury tenant | Giro's Fineract tenant (`config.CBATenantId`; [V] D1.4 §1) becomes a dedicated `giro` (treasury) tenant. It holds investor clients and fixed deposits, and records deployments to lender tenants through the inter-tenant service. Service client role: `SERVICE_TREASURY` |
| Noisy neighbour | Rate limit per tenant; COB partitions per tenant run at bounded concurrency |

## 5. Security controls (summary)

| Control | Implementation |
|---|---|
| Transport | TLS 1.2+ everywhere, HSTS. No `rejectUnauthorized:false` in clients |
| Secrets | From a secret manager or env injection. **Boot fails if a required secret is missing.** No default credentials (fixes DS-02) |
| Input validation | zod schemas on every route (body, query, params). Reject unknown fields on the native API |
| Output | No stack traces. `problem+json` with `requestId` |
| Logging | Pino/winston redaction of `authorization`, `cookie`, `password`, `bvn`, `nin`, `accountNumber` (fixes DS-01) |
| PII | BVN and NIN encrypted (AES-256-GCM, KMS-wrapped data key) plus HMAC hash for lookup. Phone and email in clear (needed for search) but excluded from logs |
| Webhooks | HMAC-SHA256 signature (`X-FM-Signature: t=<ts>,v1=<hex>`), 5 min timestamp tolerance, per-subscription secret rotation |
| Admin surfaces | Bull Board and MCP are disabled in production, or exposed only behind VPN with `PLATFORM_ADMIN` (ADR-008) |
| Dependency hygiene | `npm audit` in CI, Renovate, SBOM (CycloneDX) per build |
| Pen-test | Before the first production tenant cut-over (gate G-SEC in the roadmap) |
