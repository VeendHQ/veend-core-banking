# Module: Integrations Boundary (payments, LendLot-compat facade, external services)

> Phase: P5 (compat facade), P1 (payment types) · Priority: P0 for the compat facade · Status: Spec draft
> Fineract reference: n/a (compatibility target is Fineract's own REST API)
> veend-tenant touchpoints: all of `services/lendlot` (via compat); Remita, Giro, IPPIS, Paystack, Flutterwave, CoralPay, Cgate, OnePipe, Vida, LoanDisk, Stanbic integrations stay in veend-tenant ([V] D1.2 §2)

## 1. Overview & business purpose

> **ADR-010 update:** the "CBA does not move money" boundary below applies to LENDER and TREASURY tenants. BANK tenants move money inside Fineract-Mod through [payments-transfers-bills](payments-transfers-bills.md), [teller-branch-cash](teller-branch-cash.md) and [card-management](card-management.md).

This module sets the boundary between Fineract-Mod and the outside world:
1. **What the CBA does not do:** move money, talk to payment rails, run credit decisioning, or send borrower notifications. Those stay in veend-tenant, which already integrates about 15 providers.
2. **How rails are represented in the CBA:** payment types, clearing accounts and external references.
3. **The LendLot-compat facade** (ADR-007), the temporary adapter that lets veend-tenant switch CBA by configuration.
4. **Future direct integrations** (credit bureau reporting, NIBSS, card schemes): extension points only.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-INT-01 | Every financial command accepts `paymentType` and `externalRef` (provider reference) and persists them. `(paymentType, externalRef)` is unique (BR-TXN-01) |
| FR-INT-02 | Per-rail clearing account mapping (organisation module) so finance can reconcile settlement statements per rail |
| FR-INT-03 | **Compat facade**: serves every live endpoint of D2.2 §2 with request and response compatibility for the fields listed in D2.2 §3–4 |
| FR-INT-04 | Compat auth: Basic → service client mapping; tenant header check |
| FR-INT-05 | Compat paymentTypeId mapping: Fineract `paymentTypeId` → Fineract-Mod payment type per tenant (migrated `m_payment_type` ids). veend-tenant's hard-coded `1` maps to the migrated type 1 [A: likely "Money Transfer"/"Cash"] |
| FR-INT-06 | Compat derived idempotency (api-design §4) |
| FR-INT-07 | Compat error shape: `{developerMessage, httpStatusCode, defaultUserMessage, userMessageGlobalisationCode, errors:[{developerMessage, defaultUserMessage, userMessageGlobalisationCode, parameterName}]}` |
| FR-INT-08 | Compat traffic metrics per endpoint and tenant, plus a "last used" timestamp (to know when the facade can be retired) |
| FR-INT-09 | Credit-bureau reporting extract (CRC, FirstCentral, CreditRegistry formats): **in scope, P7-G** via [regulatory-reporting](regulatory-reporting.md) |

## 3. Features

| Feature | MVP? |
|---|---|
| Payment types + clearing | Yes |
| Compat facade: clients (C1–C4), products (P1–P6), loans (L1–L10), transactions (T1–T2), reports (R1–R3) | Yes |
| Compat: write-off and recovery commands (for M3) | Yes |
| Native client SDK (`@veend/cba-client`, TypeScript) for veend-tenant M1+ | Yes (P5) |
| Bureau reporting, NIBSS direct debit, card settlement | Future |

## 4. Business rules

- **BR-INT-01.** The facade contains **no business logic**. It maps compat requests to native commands and native results to compat responses.
- **BR-INT-02.** Unknown compat fields are ignored and logged (Fineract is lenient). Known-wrong payloads (DF-08) are rejected.
- **BR-INT-03.** Compat dates are parsed with `dateFormat` and `locale` from the body (`dd MMMM yyyy`, `en`). They are interpreted as **tenant-local dates**, not UTC.
- **BR-INT-04.** Compat numbers are emitted as JSON numbers with currency scale (`15500` or `15500.5`), matching Fineract.

## 5. Data model

```sql
CREATE TABLE compat_payment_type_map (fineract_payment_type_id BIGINT PRIMARY KEY, payment_type_id BIGINT NOT NULL REFERENCES payment_type(id));
CREATE TABLE compat_usage (route TEXT, method TEXT, day DATE, calls INT, PRIMARY KEY (route, method, day));
CREATE TABLE compat_derived_key (key_hash CHAR(64) PRIMARY KEY, txn_id BIGINT, created_at TIMESTAMPTZ DEFAULT now());
```

## 6. API endpoints

- Compat routes under `/fineract-provider/api/v1/` (see D2.2 §2).
- Native: `GET/POST /v1/payment-types` (organisation).

## 7. Service-layer responsibilities

`compat/lendlot/routes/*` (one file per Fineract resource), `compat/lendlot/mappers/*` (request → command, result → Fineract JSON), `compat/lendlot/errors.ts`, `compat/lendlot/dates.ts` (`[y,m,d]` arrays and the `dd MMMM yyyy` parser), `compat/lendlot/reports/*`.

## 8. Dependencies

All domain modules (via commands and queries only).

## 9. Integration requirements (summary of external systems)

| System | Owner after migration | CBA interaction |
|---|---|---|
| Giro (disbursement, wallet, payment requests) | veend-tenant | payment type GIRO; disbursement `externalRef` = Giro transfer id |
| Remita (mandates, deductions) | veend-tenant | payment type REMITA; ref = `${mandateRef}:${deductionDate}` |
| IPPIS | veend-tenant | payment type IPPIS |
| Paystack, Flutterwave, CoralPay, Cgate | veend-tenant | per-rail payment types; ref = provider reference |
| OnePipe, Vida | veend-tenant | payment types ONEPIPE, VIDA |
| LoanDisk | veend-tenant ([A] decommission?) | none |
| Insurance | veend-tenant | payment type INSURANCE for recoveries |
| **Giro FD funding and investor payouts** | Giro (`bill-payment-portal`) | payment type GIRO; refs = Giro purchase or payment-request references. A second compat consumer (savings endpoints, D2.7 §4) |

## 10. Authorization

Compat routes require the `SERVICE_LENDING` role permissions equivalent to the native operations.

## 11. Transactional & accounting implications

Clearing accounts make settlement reconciliation possible: Σ CBA postings to `REMITA_CLEARING` for a day must equal the Remita settlement amount (ops report `recon.clearing_account`).

## 12. Edge cases

- veend-tenant sends `transactionDate` = today for old deductions (DF-01). The facade cannot know the true date, so it accepts the value. The value-date fix lives in veend-tenant (M2).
- Shared mutable body leakage from the old client (DS-05) can add stray query params to GETs. The facade ignores unknown query params.

## 13. Testing

- **Contract tests:** at least 200 recorded LendLot request/response pairs per endpoint, scrubbed from the `APICalls` collection. The facade response must match on the consumed fields (D2.2 §4) and on status code.
- **Shadow comparison** during the parallel run (D6).

## 14. Status & gaps

- Not implemented.
- [A] Production `m_payment_type` list.
- [A] The production Fineract version, which determines the exact JSON shapes (some field names changed across releases, e.g. `transactionProcessingStrategyId` → `…Code`).
