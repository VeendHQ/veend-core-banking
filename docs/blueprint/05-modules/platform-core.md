# Module: Platform Core

> Phase: P0 (foundations) and P1 (maker-checker UI hooks) · Priority: P0 · Status: Spec draft
> Fineract reference: `fineract-core` (commands, tenants, business date, useradministration), `fineract-security`, `fineract-command*`
> veend-tenant touchpoints: none directly. veend-tenant authenticates as a service client.

## 1. Overview & business purpose

This module provides the cross-cutting capabilities every financial module relies on:
- **tenancy**: resolution, isolation and provisioning
- **identity and RBAC**: users, service clients, roles and permissions
- the **command bus**: validation, idempotency, maker-checker, transaction scope and audit
- **business date** and tenant configuration
- the **outbox** (shared with [events-notifications](events-notifications.md))

It owns the platform schema and the `command_log`, `entity_audit`, `idempotency_key`, `business_date`, `app_user`, `role`, `permission` and `maker_checker_config` tables in each tenant schema.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-PLT-01 | Provision a tenant idempotently: schema, DB role, migrations, CoA template, default roles, admin user, sandbox twin |
| FR-PLT-02 | Authenticate service clients (OAuth2 client credentials) and human users (OIDC + MFA). Compat Basic auth on compat routes only |
| FR-PLT-03 | Resolve the tenant only from the principal. Reject a mismatching `Fineract-Platform-TenantId` header |
| FR-PLT-04 | Authorise every command and query against permission codes in the `ACTION_ENTITY` form |
| FR-PLT-05 | Enforce idempotency on every money-moving or creating command (`Idempotency-Key` or derived) |
| FR-PLT-06 | Hold commands for approval when maker-checker is enabled for the permission (optionally above an amount threshold). Approve or reject by a different user. Expire after the TTL |
| FR-PLT-07 | Record every command (success, failure, pending, rejected) in a hash-chained `command_log`, plus entity before/after diffs |
| FR-PLT-08 | Maintain a per-tenant business date and COB date. Allow manual advance for operations (with permission) and an automatic daily roll in the tenant timezone |
| FR-PLT-09 | Expose a typed tenant configuration (timezone, currency, rounding default, max backdate days, COB schedule, reversal window) |
| FR-PLT-10 | Provide health and readiness checks, and a migration-status endpoint |
| FR-PLT-11 | **Transaction limits** (RT-10): per role and permission, a maximum amount per command and per business day. A command above the limit is rejected (`422 LIMIT_EXCEEDED`), or routed to maker-checker when configured. Limits are stored in `transaction_limit(role_id, permission_code, per_txn_max, per_day_max)` |

## 3. Features & sub-features

| Feature | Sub-feature | MVP? | Notes |
|---|---|---|---|
| Tenancy | Registry, provisioning, schema migration runner, sandbox twin, suspend/close | Yes | ADR-003 |
| IAM | Service clients (secret rotation), users, roles, permissions, MFA, password policy, lockout | Yes | Users may also come from an IdP |
| | Office-scoped data access | Yes | Hierarchy path filter |
| Command bus | Validation, idempotency, authz, transaction, audit, outbox hook | Yes | ADR-005 |
| Maker-checker | Per permission, thresholds, inbox, approve/reject, expiry | Yes | |
| Business date | Get, advance, auto-roll | Yes | |
| Config | Typed config with versioned history | Yes | |
| Global configuration flags (Fineract `c_configuration`) | Subset only | Partial | Only the flags we use |

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-PLT-01 | The tenant key matches `^[a-z][a-z0-9_]{2,30}$`. The schema is `t_<key>` and is immutable after creation |
| BR-PLT-02 | An idempotency record is `(tenant, client_id, key)`. A replay with the same `request_hash` returns the stored status and body. A different hash returns 409. A record is written **only on commit** |
| BR-PLT-03 | The maker cannot be the checker. A checker needs `<PERMISSION>_CHECKER`. The approval re-validates against current state |
| BR-PLT-04 | A pending command expires after `maker_checker_ttl_hours` (default 72) and becomes EXPIRED |
| BR-PLT-05 | The business date never goes backwards. Advancing more than 1 day queues COB for each skipped date in order |
| BR-PLT-06 | `command_log.hash = sha256(prev_hash ‖ id ‖ type ‖ payload_hash ‖ status ‖ finished_at)`. The genesis row has `prev_hash = sha256(tenant_key)` |
| BR-PLT-07 | Service-client tokens last 10 minutes. Client secrets are stored as argon2id hashes and rotated with 2 active secrets |
| BR-PLT-08 | A human user locks after 5 failed logins in 15 minutes |

## 5. Data model

```sql
-- platform schema
CREATE TABLE platform.tenant (key TEXT PRIMARY KEY, schema TEXT UNIQUE NOT NULL, name TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('PROVISIONING','ACTIVE','SUSPENDED','CLOSED')),
  timezone TEXT NOT NULL DEFAULT 'Africa/Lagos', base_currency CHAR(3) NOT NULL DEFAULT 'NGN',
  sandbox_of TEXT REFERENCES platform.tenant(key), config JSONB NOT NULL DEFAULT '{}', created_at TIMESTAMPTZ DEFAULT now());
CREATE TABLE platform.oauth_client (client_id TEXT PRIMARY KEY, tenant_key TEXT REFERENCES platform.tenant(key),
  secret_hashes TEXT[] NOT NULL, scopes TEXT[] NOT NULL, legacy_basic_username TEXT UNIQUE, ip_allowlist CIDR[], active BOOLEAN DEFAULT true);
CREATE TABLE platform.schema_migration_status (tenant_key TEXT, version TEXT, applied_at TIMESTAMPTZ, PRIMARY KEY (tenant_key, version));

-- tenant schema
CREATE TABLE app_user (id BIGSERIAL PRIMARY KEY, username TEXT UNIQUE NOT NULL, email TEXT, office_id BIGINT,
  idp_subject TEXT UNIQUE, password_hash TEXT, mfa_secret_enc BYTEA, status TEXT NOT NULL, failed_logins INT DEFAULT 0);
CREATE TABLE role (id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE NOT NULL, description TEXT, disabled BOOLEAN DEFAULT false);
CREATE TABLE permission (code TEXT PRIMARY KEY, grouping TEXT, entity TEXT, action TEXT, can_maker_checker BOOLEAN NOT NULL);
CREATE TABLE role_permission (role_id BIGINT REFERENCES role(id), permission_code TEXT REFERENCES permission(code), PRIMARY KEY (role_id, permission_code));
CREATE TABLE principal_role (principal_type TEXT, principal_id TEXT, role_id BIGINT REFERENCES role(id), PRIMARY KEY (principal_type, principal_id, role_id));
CREATE TABLE maker_checker_config (permission_code TEXT PRIMARY KEY REFERENCES permission(code), enabled BOOLEAN NOT NULL, threshold_amount NUMERIC(19,6));
CREATE TABLE command_log (id BIGSERIAL PRIMARY KEY, type TEXT NOT NULL, permission_code TEXT NOT NULL,
  entity TEXT, entity_id BIGINT, actor_type TEXT, actor_id TEXT, maker_id TEXT, checker_id TEXT,
  status TEXT NOT NULL CHECK (status IN ('PENDING_APPROVAL','RUNNING','SUCCEEDED','FAILED','REJECTED','EXPIRED')),
  payload JSONB NOT NULL, payload_hash TEXT NOT NULL, idempotency_key TEXT, result JSONB, error JSONB,
  submitted_at TIMESTAMPTZ NOT NULL DEFAULT now(), approved_at TIMESTAMPTZ, finished_at TIMESTAMPTZ,
  prev_hash TEXT, hash TEXT);
CREATE INDEX ON command_log (entity, entity_id);
CREATE INDEX ON command_log (status) WHERE status = 'PENDING_APPROVAL';
CREATE TABLE entity_audit (id BIGSERIAL PRIMARY KEY, command_id BIGINT REFERENCES command_log(id), entity TEXT, entity_id BIGINT, before JSONB, after JSONB);
CREATE TABLE idempotency_key (client_id TEXT, key TEXT, request_hash TEXT NOT NULL, status_code INT NOT NULL, response JSONB NOT NULL,
  command_id BIGINT, created_at TIMESTAMPTZ DEFAULT now(), PRIMARY KEY (client_id, key));
CREATE TABLE business_date (type TEXT PRIMARY KEY CHECK (type IN ('BUSINESS','COB')), date DATE NOT NULL, updated_at TIMESTAMPTZ);
CREATE TABLE tenant_config (key TEXT PRIMARY KEY, value JSONB NOT NULL, version INT NOT NULL, updated_by TEXT, updated_at TIMESTAMPTZ);
```

## 6. API endpoints

| Method & path | Permission | Notes |
|---|---|---|
| `POST /platform/tenants` | PLATFORM_ADMIN | Body `{key, name, timezone, baseCurrency, coaTemplate}`. Idempotent on `key` |
| `POST /oauth/token` | — | client_credentials |
| `GET /v1/me` | any | principal, tenant, permissions |
| `GET/POST/PATCH /v1/users`, `/v1/roles`, `GET /v1/permissions`, `PUT /v1/roles/{id}/permissions` | `*_USER`, `*_ROLE` | |
| `GET /v1/maker-checker/config`, `PUT …/{permission}` | `UPDATE_MAKERCHECKER` | |
| `GET /v1/commands?status=PENDING_APPROVAL` | `READ_MAKERCHECKER` | checker inbox |
| `POST /v1/commands/{id}/approve` / `reject` | `<perm>_CHECKER` | |
| `GET /v1/business-date`, `POST /v1/business-date/advance {to}` | `READ/UPDATE_BUSINESS_DATE` | |
| `GET/PUT /v1/config/{key}` | `READ/UPDATE_CONFIGURATION` | |
| `GET /v1/audit?entity&id` | `READ_AUDIT` | |
| `GET /health/live`, `/health/ready` | — | |

## 7. Service-layer responsibilities

- `TenantProvisioner`: creates the schema and role, runs migrations, seeds data.
- `TenantContext`: AsyncLocalStorage holding `{tenantKey, principal, businessDate, cobDate, txn}`.
- `CommandBus.execute`, `CommandRegistry`: each entry is `{type, schema, permission, makerChecker, amountOf?, idempotency:'required'|'derived'|'none', handler}`.
- `IdempotencyStore`, `MakerCheckerService`, `AuditWriter` (hash chain), `BusinessDateService`, `ConfigService`.
- `AuthMiddleware`: JWT verification, compat Basic auth mapping, tenant-header check.

## 8. Dependencies

- Upstream: Postgres, Redis, KMS/secret manager.
- Downstream: every module.

## 9. Integration requirements

- OIDC IdP (configurable).
- The outbox shares the transaction.
- A metrics exporter.

## 10. Authorization & permissions

Seed permissions mirror Fineract codes for the entities in scope: `CLIENT`, `LOAN`, `LOANPRODUCT`, `CHARGE`, `GLACCOUNT`, `JOURNALENTRY`, `GLCLOSURE`, `PROVISIONING`, `OFFICE`, `STAFF`, `PAYMENTTYPE`, `USER`, `ROLE`, `REPORT`, `BUSINESS_DATE`, `MAKERCHECKER`, `CONFIGURATION`, `AUDIT`, `WEBHOOK`. Each has `CREATE_`, `UPDATE_`, `DELETE_`, `READ_`, and domain actions (for example `APPROVE_LOAN`, `DISBURSE_LOAN`, `REPAYMENT_LOAN`, `ADJUST_LOAN`, `WRITEOFF_LOAN`, `UNDOAPPROVAL_LOAN`, `REJECT_LOAN`) and `_CHECKER` variants.

## 11. Transactional & accounting implications

- This module defines the transaction boundary for all modules (one command, one transaction).
- It does not post journals itself.

## 12. Error handling & edge cases

| Case | Behaviour |
|---|---|
| Idempotent replay while the original is still running | The second request waits on an advisory lock `pg_advisory_xact_lock(hash(client,key))` up to 5 s, then replays or returns 409 `IN_PROGRESS` |
| Approval after the underlying state changed | Re-validation fails, the command becomes FAILED with the reason, and the maker is notified |
| Business-date advance while COB is still running for the previous date | Rejected (409) |
| Tenant suspended | All non-read commands return 403 `TENANT_SUSPENDED` |
| Audit write fails | The command fails; audit is never skipped |

## 13. Testing requirements

- **Unit:** permission resolution, hash chain, idempotency hashing (canonical JSON).
- **Property tests:** N concurrent identical requests produce exactly one execution and N identical responses.
- **Integration:** cross-tenant isolation (a token for tenant A gets 404 or 403 on B's ids; a raw SQL attempt on B's schema under A's role fails); maker-checker full cycle; expiry.
- **Security:** JWT tampering, alg confusion, expired tokens, Basic auth only on compat routes.

## 14. Implementation status & gaps

- Status: **not implemented**. The framework has none of this ([V] D1.3 G6, G9).
- Open: [A] the IdP choice, the tenant count, and whether any lender requires a dedicated DB (handled by ADR-003's escape hatch).
