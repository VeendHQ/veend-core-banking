# Module: Clients

> Phase: P1 · Priority: P0 (parity) · Status: Spec draft
> Fineract reference: `ClientsApiResource`, `m_client`, datatable `userdata`, `SearchApiResource`
> veend-tenant touchpoints: `createBorrower` (search by phone then create), `getBorrower`, `updateBorrower` (+ `userdata.email`) ([V] D2.2 C1–C4). `Users.clientId` holds the id.

## 1. Overview & business purpose

This module is the CBA party record for each borrower: identity, contact details, status and KYC references. Full KYC evidence (bank statements, BVN verification payloads) **stays in veend-tenant**. The CBA keeps the identifiers needed for regulatory reporting and de-duplication.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-CLI-01 | Create a client with `firstname`, `lastname`, optional `middlename`, `mobileNo`, `email`, `dateOfBirth`, `gender`, `officeId`, `externalId`, and `legalForm` (PERSON/ENTITY). Activate immediately or later |
| FR-CLI-02 | Store a BVN and NIN encrypted, plus `bvn_hash` for lookup. Validate that a BVN is 11 digits |
| FR-CLI-03 | Search by `mobileNo` (exact, normalised to E.164), `externalId`, `bvnHash`, `accountNo`, or a name prefix |
| FR-CLI-04 | Update contact and name fields with an audit diff. Changing the BVN requires maker-checker |
| FR-CLI-05 | Status lifecycle: PENDING(100) → ACTIVE(300) → CLOSED(600). Also REJECTED and WITHDRAWN (Fineract codes) |
| FR-CLI-06 | Enforce a de-duplication policy: by default, unique active `mobileNo` per tenant **and** unique `bvn_hash` per tenant |
| FR-CLI-07 | Provide the compat endpoints C1–C4 with Fineract shapes |
| FR-CLI-08 | Provide a client summary: active loans, total outstanding, arrears (a read model for veend-tenant) |

## 3. Features

| Feature | Sub-feature | MVP? |
|---|---|---|
| Client CRUD | Create/activate, update, close | Yes |
| Identifiers | BVN, NIN, phone, email; encrypted storage | Yes |
| Search & dedupe | Phone, BVN hash, external id | Yes |
| Custom attributes | Replaces `userdata` datatable (`email` and any extras) | Yes (see [extensibility](extensibility-datatables.md)) |
| Client charges | Fineract client-level fees | No (future) |
| Family members, addresses, documents | | No (future); documents stay in veend-tenant |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-CLI-01 | `mobileNo` is normalised: strip spaces, convert a leading `0` to `+234`. Stored as E.164 |
| BR-CLI-02 | When creating, if an ACTIVE client exists with the same normalised `mobileNo` **or** `bvn_hash`, return **the existing client** with `200` and `"deduplicated": true` on the compat API. This matches the current `createBorrower` search-then-create behaviour ([V] `lendlot/index.js:515-525`). The native API returns 409 `CLIENT_EXISTS` with the existing id |
| BR-CLI-03 | `activationDate` ≥ the office opening date and ≤ the business date |
| BR-CLI-04 | A client cannot be closed while it has loans in status 100/200/300/700 |
| BR-CLI-05 | `externalId` is unique per tenant. veend-tenant should send `Users._id` (M2) |
| BR-CLI-06 | The account number is generated as a zero-padded 9-digit `id` (Fineract default; [A] confirm the production format) |

## 5. Data model

```sql
CREATE TABLE client (
  id BIGSERIAL PRIMARY KEY, account_no VARCHAR(20) UNIQUE NOT NULL, external_id VARCHAR(100) UNIQUE,
  legacy_fineract_id BIGINT UNIQUE,
  office_id BIGINT NOT NULL REFERENCES office(id), staff_id BIGINT REFERENCES staff(id),
  status SMALLINT NOT NULL, legal_form SMALLINT NOT NULL DEFAULT 1,
  firstname TEXT, middlename TEXT, lastname TEXT, fullname TEXT, display_name TEXT NOT NULL,
  mobile_no VARCHAR(20), email CITEXT, date_of_birth DATE, gender TEXT,
  bvn_enc BYTEA, bvn_hash CHAR(64), nin_enc BYTEA, nin_hash CHAR(64),
  activation_date DATE, closed_on DATE, submitted_on DATE NOT NULL,
  attributes JSONB NOT NULL DEFAULT '{}', version INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(), updated_at TIMESTAMPTZ DEFAULT now());
CREATE UNIQUE INDEX uq_client_mobile_active ON client (mobile_no) WHERE status = 300;
CREATE UNIQUE INDEX uq_client_bvn_active ON client (bvn_hash) WHERE status = 300 AND bvn_hash IS NOT NULL;
CREATE INDEX ix_client_name ON client (lower(display_name) text_pattern_ops);
```

## 6. API endpoints

**Native:**
- `POST /v1/clients` (Idempotency-Key)
- `GET /v1/clients/{id}`
- `GET /v1/clients?mobileNo=&externalId=&bvn=`: the `bvn` value is hashed server-side and never logged.
- `PATCH /v1/clients/{id}`
- `POST /v1/clients/{id}/activation`, `/closure`
- `GET /v1/clients/{id}/summary`

**Compat:**
- **C1** `GET /search?resource=clients,...&query=` returns `[{entityId, entityType:'CLIENT', entityMobileNo, entityName, entityAccountNo}]`.
- **C2** `POST /clients` returns `{officeId, clientId, resourceId}`. It accepts the `datatables:[{registeredTableName:'userdata', data:{email}}]` payload and maps it to `client.email`.
- **C3** `GET /clients/{id}` returns the Fineract client shape (`id, accountNo, status{id,code,value}, active, activationDate[y,m,d], firstname, lastname, displayName, mobileNo, officeId, officeName, timeline{…}`).
- **C4** `PUT /clients/{id}` returns `{clientId, resourceId, changes}`. `PUT /datatables/userdata/{id}` maps to the email and attributes update.

## 7. Service-layer responsibilities

- Commands: `CreateClient`, `ActivateClient`, `UpdateClient`, `CloseClient`.
- Queries: `FindClients`, `GetClient`, `ClientSummary`.
- `PiiCipher`: KMS envelope encryption, plus HMAC with a tenant-specific key for hashes.

## 8. Dependencies

- Upstream: platform-core, organisation.
- Downstream: loan-lifecycle (borrower), reporting.

## 9. Integration requirements

- Events: `client.created`, `client.updated`, `client.closed`.
- No external calls. BVN verification stays in veend-tenant.

## 10. Authorization

`CREATE_CLIENT`, `ACTIVATE_CLIENT`, `UPDATE_CLIENT`, `CLOSE_CLIENT`, `READ_CLIENT`, `READ_CLIENT_PII` (needed to see a decrypted BVN or NIN; every such read is logged). Maker-checker is recommended for `UPDATE_CLIENT` when the BVN changes.

## 11. Transactional & accounting implications

None. Clients do not post to the GL.

## 12. Error handling & edge cases

| Case | Behaviour |
|---|---|
| Race: two creates with the same phone | The unique partial index makes one fail with a constraint violation. The handler catches it and returns the existing client (compat) or 409 (native) |
| Client exists in another tenant (cross-lender borrower) | Allowed. Tenants are isolated |
| Migrated clients with duplicate phones | Migration quarantines them, and the uniqueness index is created after cleanup. [A] Duplicate volume unknown |
| Phone number recycled by the telco | The operator closes the old client or changes its number (maker-checker) |

## 13. Testing

- Unit: phone normalisation table tests; dedupe policy.
- Integration: compat C1–C4 contract tests from recorded LendLot samples; concurrency test on dedupe.
- Security: BVN never appears in logs (log-capture assertion).

## 14. Status & gaps

- Not implemented.
- [A] Production rule for duplicate phones across inactive clients.
- [A] Whether `userdata` holds more columns than `email` in production (inspect `x_registered_table`).
