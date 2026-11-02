# D3.3 API Design and Versioning

> [R] throughout.

## 1. Two API surfaces

| Surface | Base path | Consumers | Lifetime |
|---|---|---|---|
| **LendLot-compat** | `/fineract-provider/api/v1/...` | veend-tenant (unchanged client, phase M0) | Until veend-tenant migrates to the native API (M5). Then deprecated with 6 months' notice |
| **Native** | `/v1/...` | veend-tenant (new `CbaClient`), back-office, future partners | Long-term |

Both surfaces dispatch the **same commands** (see [processing](processing.md)). The compat layer is a pure translator ([ADR-007](adr/ADR-007-lendlot-compat-facade.md)).

## 2. Native API conventions

| Concern | Convention |
|---|---|
| Style | Resource-oriented REST. State transitions are sub-resources: `POST /v1/loans/{id}/approval`, `/disbursements`, `/repayments` |
| Versioning | URI major version (`/v1`). Additive changes only within a version. A breaking change means `/v2` side by side, with at least 6 months' overlap. The `Sunset` and `Deprecation` headers are set on deprecated routes. Fixes the framework's contiguous-version loader bug (G15) |
| Media | `application/json; charset=utf-8`. Amounts are decimal **strings**; dates are `YYYY-MM-DD`; timestamps are RFC 3339 UTC |
| IDs | Numeric internal `id` plus `accountNo`; callers may supply `externalId` (unique per tenant) and fetch by it: `GET /v1/loans/external/{externalId}` |
| Idempotency | **Required** on every `POST` that creates or moves money: the `Idempotency-Key` header (max 120 chars). Same key + same body hash returns the stored response with the original status. Same key + different body returns `409 IDEMPOTENCY_KEY_REUSED`. Keys are retained 30 days, and money-moving `externalRef` values forever (DB unique constraint) |
| Concurrency | `ETag` on single-resource GET (`W/"<version>"`). `If-Match` is optional on `PATCH`; a mismatch returns `412` |
| Pagination | Cursor based: `?limit=50&cursor=<opaque>` returns `{data, nextCursor}`; `limit` max 200 |
| Filtering | Explicit query params per resource (no arbitrary `where`, which fixes G7) |
| Errors | RFC 9457 `application/problem+json`: `{type, title, status, code, detail, errors:[{field, code, message}], requestId}` |
| Maker-checker | A command that needs approval returns `202 Accepted` `{commandId, status:"PENDING_APPROVAL"}`. The result is fetched via `GET /v1/commands/{id}` |
| Tracing | Accepts and echoes `X-Request-Id` and W3C `traceparent` |
| Rate limits | Per OAuth client and tenant. `429` with `Retry-After` |

### Error code catalogue (excerpt)

| HTTP | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Schema (zod) failure |
| 401 / 403 | `UNAUTHENTICATED` / `PERMISSION_DENIED` | |
| 404 | `NOT_FOUND` | Resource not found in the caller's tenant (cross-tenant ids also return 404) |
| 409 | `INVALID_STATE` | e.g. disbursing a loan that isn't APPROVED |
| 409 | `IDEMPOTENCY_KEY_REUSED`, `DUPLICATE_EXTERNAL_REF` | |
| 409 | `CONCURRENT_MODIFICATION` | Lock timeout |
| 422 | `BUSINESS_RULE_VIOLATION` | e.g. principal outside the product range, value date in a closed period, value date in the future |
| 423 | `PERIOD_CLOSED` | Posting into a closed GL period |
| 503 | `COB_IN_PROGRESS` | Loan locked by its COB partition (retry after) |

## 3. Native endpoint catalogue (MVP)

| Module | Endpoints |
|---|---|
| Platform | `GET /v1/me`, `GET /v1/business-date`, `POST /v1/business-date/advance` (ops), `GET /v1/commands/{id}`, `POST /v1/commands/{id}/approve`, `/reject`, `GET /v1/audit?entity=&id=` |
| Organisation | `GET/POST /v1/offices`, `GET/POST /v1/staff`, `GET/POST /v1/payment-types`, `GET/POST /v1/holidays`, `GET/PUT /v1/working-days`, `GET /v1/currencies` |
| Clients | `POST /v1/clients`, `GET /v1/clients/{id}`, `GET /v1/clients?mobileNo=&externalId=&bvnHash=`, `PATCH /v1/clients/{id}`, `POST /v1/clients/{id}/activation`, `/closure` |
| Products | `GET/POST /v1/loan-products`, `GET/PATCH /v1/loan-products/{id}`, `GET /v1/loan-products/{id}/template?clientId=` |
| Charges | `GET/POST /v1/charges`, `PATCH/DELETE /v1/charges/{id}`, `POST /v1/loans/{id}/charges`, `POST /v1/loans/{id}/charges/{chargeId}/waiver` |
| Schedule | `POST /v1/loan-schedules/calculate` (stateless) |
| Loans | `POST /v1/loans` (submit), `POST /v1/loans/book` (composite: submit + approve [+ disburse], one command), `GET /v1/loans/{id}?include=schedule,transactions,charges`, `POST /v1/loans/{id}/approval`, `DELETE /v1/loans/{id}/approval` (undo), `POST /v1/loans/{id}/rejection`, `POST /v1/loans/{id}/withdrawal`, `POST /v1/loans/{id}/disbursements`, `DELETE /v1/loans/{id}/disbursements/last` (undo) |
| Transactions | `POST /v1/loans/{id}/repayments`, `POST /v1/loans/{id}/transactions/{txId}/reversal`, `POST /v1/loans/{id}/transactions/reversal-by-ref` (`{paymentType, externalRef}`), `GET /v1/loans/{id}/transactions`, `GET /v1/loans/{id}/payoff-quote?date=` |
| Credit risk | `POST /v1/loans/{id}/write-off`, `DELETE /v1/loans/{id}/write-off` (undo), `POST /v1/loans/{id}/recoveries`, `POST /v1/loans/{id}/waivers/interest` |
| Accounting | `GET/POST /v1/gl-accounts`, `GET /v1/journal-entries?…`, `POST /v1/journal-entries` (manual, maker-checker), `POST /v1/journal-entries/{id}/reversal`, `GET/POST /v1/gl-closures`, `GET/PUT /v1/loan-products/{id}/gl-mappings` |
| Provisioning | `GET/POST /v1/provisioning-criteria`, `POST /v1/provisioning-entries` (run), `GET /v1/provisioning-entries/{id}` |
| Fixed deposits | `GET/POST /v1/deposit-products`, `POST /v1/deposit-accounts`, `GET /v1/deposit-accounts/{id}`, `/deposits`, `/deployments`, `/payouts/{seq}/settlement`, `/premature-closure`, `/summary`, `GET /v1/deposit-payouts?status=DUE` (see [fixed-deposits §6](../05-modules/fixed-deposits.md#6-api-endpoints)) |
| Reports | `GET /v1/reports` (catalogue), `GET /v1/reports/{code}?params` (JSON or CSV via `Accept`) |
| Webhooks | `GET/POST /v1/webhook-subscriptions`, `POST /v1/webhook-subscriptions/{id}/test`, `GET /v1/webhook-deliveries?status=failed`, `POST /v1/webhook-deliveries/{id}/redeliver` |

### Example: repayment

```http
POST /v1/loans/48211/repayments
Authorization: Bearer <token>
Idempotency-Key: remita:48211:2026-10-28:REM-889912
Content-Type: application/json

{ "amount": "15500.00", "valueDate": "2026-10-28", "paymentType": "REMITA",
  "externalRef": "REM-889912", "note": "Payroll deduction Oct-2026" }
```
```http
201 Created
{ "data": {
    "transactionId": 990122, "loanId": 48211, "type": "REPAYMENT",
    "valueDate": "2026-10-28", "bookingDate": "2026-11-02", "amount": "15500.00",
    "allocation": { "principal": "12000.00", "interest": "3000.00", "fees": "500.00", "penalties": "0.00", "overpayment": "0.00" },
    "loan": { "status": "ACTIVE", "statusCode": 300, "totalOutstanding": "31000.00" },
    "journalEntryId": 771203 },
  "requestId": "…" }
```

## 4. LendLot-compat contract

The compat facade serves exactly the live endpoints in the [inventory §2](../02-current-state/lendlot-endpoint-inventory.md#2-live-endpoints-must-be-served-by-the-fineract-mod-lendlot-compat-facade) with Fineract-shaped requests and responses:

1. **Auth:** it accepts Basic auth, mapped to a tenant-scoped service client. This is kept only for phase M0. Credentials are stored as argon2 hashes, with a rate limit and an IP allowlist.
2. **Tenant:** taken from the `Fineract-Platform-TenantId` header, which must match the principal's tenant.
3. **Shapes:** Fineract response shapes are reproduced, including `[yyyy,m,d]` date arrays, `status.{active, closed, overpaid, pendingApproval, waitingForDisbursal}` booleans, `changes`, `resourceId`/`loanId`/`clientId`, and positional report rows.
4. **Idempotency:** the facade derives an idempotency key when none is supplied:
   - repayments use `sha256(loanId|amount|transactionDate|note)` within a 24 h window;
   - the native `externalRef` uniqueness applies once veend-tenant sends `externalId` (M2).
5. **Date handling:** honours `transactionDate`/`actualDisbursementDate`/`approvedOnDate` in Fineract format (today veend-tenant always sends today; DF-01 is fixed client-side in M2).
6. **Strictness:** unknown fields are logged and ignored, as Fineract does with lenient parsing. Known-bad payloads (DF-08) are rejected with Fineract-style `400` errors (`{developerMessage, errors:[{userMessageGlobalisationCode}]}`).
7. **Contract tests:** recorded request/response pairs from LendLot production traffic (`APICalls` collection, scrubbed) replayed against the facade. This is the acceptance gate for M0 (see D6).
8. **Second consumer, Giro.** The same facade also serves Giro's savings-shaped calls (G-S1..G-S6, G-C1, G-C2 in [D2.7 §4](../02-current-state/fixed-deposits-current-state.md#4-giro-cba-endpoint-inventory-compatibility-contract)) for the Giro tenant. They are backed by the fixed-deposits module (FD accounts) and a thin wallet-mirror stub (wallet savings ids and holds) until the wallet decision is made. Giro's legacy loan calls (G-L1..G-L3) map to the lending module.
