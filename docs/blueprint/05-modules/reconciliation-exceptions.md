# Module: Reconciliation and Exception Management

> Phase: P5 · Priority: P1 (P0 for go-live controls) · Status: Spec draft
> Origin: recovery-team guide §6, §10 ([input review RT-09](../09-input-reviews/recovery-team-2026-10.md#4-requirements-adopted-into-the-blueprint)); clearing-account design ([processing §3](../04-target-architecture/processing.md#3-accounting-engine)); defects DF-12, FD-09
> Fineract reference: none (Fineract has no reconciliation module)

## 1. Overview & business purpose

This module proves every day that the CBA agrees with:
- the outside world: payment providers, banks and mandates;
- itself: sub-ledgers vs GL, schedules vs transactions.

Every difference becomes an **assigned, traceable exception**, never a hidden adjustment. It also keeps the **pending/failed/reversed** lifecycle of receipts visible. The guide requires this ("a successful payment notification alone is not proof that the loan ledger was updated"), but the CBA itself only records settled money.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-REC-01 | **Receipt register**: channel systems (veend-tenant, Giro) report each inbound payment with status `INITIATED / PENDING / SUCCESS / FAILED / REVERSED / DISPUTED`, provider ref, amount, payer and intended target (loan or deposit). Idempotent on `(channel, providerRef)` |
| FR-REC-02 | **Settlement statements**: import per-rail settlement and bank statements (CSV/API: Remita, Paystack, Flutterwave, Giro, bank MT940/CSV) into `statement_line` |
| FR-REC-03 | **Matching engine** (daily, per payment type): statement line ↔ receipt register ↔ CBA transaction (`externalRef`) ↔ clearing-account journal line. Rules: exact ref, then amount + date ± tolerance window |
| FR-REC-04 | **Internal controls** (daily, after COB) per the guide's control table: (a) loan sub-ledger vs GL loan control; (b) deposit sub-ledger vs GL deposit control; (c) disbursement register vs bank/rail; (d) schedule vs ledger (paid installments, overdue amounts); (e) trial balance; (f) branch vs head office; (g) daily transaction control (initiated = posted + failed + pending + reversed) |
| FR-REC-05 | **Exceptions**: each unmatched or failed check creates an exception with a type, amount, age, owner, SLA, status (`OPEN → INVESTIGATING → RESOLVED / WRITTEN_OFF`) and links to the source records |
| FR-REC-06 | **Resolution** by (a) a linked CBA command (e.g. a missing repayment posted with its ref), (b) a corrective journal under maker-checker **linked to the exception**, or (c) an approved explanation. A closed exception can't be reopened without audit |
| FR-REC-07 | Dashboards and reports: open exceptions by age and type, clearing-account balances vs expected, a match-rate trend |

## 3. Features & sub-features

| Feature | MVP? |
|---|---|
| Receipt register (API) | Yes |
| Statement import (CSV per rail) | Yes. API connectors P9 |
| Matching (ref, amount/date) | Yes |
| Internal controls (a)–(g) | Yes ((f) only when branches exist) |
| Exception workflow + corrective journal link | Yes |
| ML/fuzzy matching | No |

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-REC-01 | A receipt in `SUCCESS` without a matching CBA transaction after T+1 business day becomes exception `UNPOSTED_RECEIPT` |
| BR-REC-02 | A CBA transaction without a statement line after the rail's settlement lag (configurable per payment type, e.g. Remita T+2) becomes `UNSETTLED_POSTING` |
| BR-REC-03 | A clearing-account GL balance ≠ Σ unmatched items becomes `CLEARING_DRIFT` (severity high) |
| BR-REC-04 | Any internal-control difference > ₦0.00 becomes an exception of severity high. A trial-balance difference is critical (P1 alert) |
| BR-REC-05 | A corrective journal must reference an exception id. A manual JE on control accounts without one is rejected |
| BR-REC-06 | `PENDING` and `FAILED` receipts never affect loan or deposit balances. They are visible in reports as such (the guide's UAT: "Pending/failed payment not falsely reported as a completed receipt") |

## 5. Data model

```sql
CREATE TABLE receipt (id BIGSERIAL PRIMARY KEY, channel TEXT NOT NULL, provider_ref TEXT NOT NULL, payment_type_id BIGINT,
  amount NUMERIC(19,6) NOT NULL, status TEXT NOT NULL, target_type TEXT, target_id BIGINT, value_date DATE,
  history JSONB NOT NULL DEFAULT '[]', cba_txn_type TEXT, cba_txn_id BIGINT, UNIQUE (channel, provider_ref));
CREATE TABLE statement_import (id BIGSERIAL PRIMARY KEY, payment_type_id BIGINT, source TEXT, period_from DATE, period_to DATE,
  file_hash CHAR(64) UNIQUE, imported_by TEXT, imported_at TIMESTAMPTZ DEFAULT now());
CREATE TABLE statement_line (id BIGSERIAL PRIMARY KEY, import_id BIGINT REFERENCES statement_import(id), value_date DATE,
  amount NUMERIC(19,6), direction CHAR(1), ref TEXT, narration TEXT, matched_receipt_id BIGINT, matched_at TIMESTAMPTZ);
CREATE TABLE recon_run (id BIGSERIAL PRIMARY KEY, run_date DATE, control TEXT, status TEXT, expected NUMERIC(19,6), actual NUMERIC(19,6), details JSONB);
CREATE TABLE recon_exception (id BIGSERIAL PRIMARY KEY, type TEXT NOT NULL, severity TEXT NOT NULL, amount NUMERIC(19,6),
  opened_on DATE NOT NULL, owner TEXT, sla_due DATE, status TEXT NOT NULL, links JSONB NOT NULL,
  resolution TEXT, resolution_command_id BIGINT, resolution_journal_id BIGINT, closed_on DATE, closed_by TEXT);
```

## 6. API endpoints

- `POST /v1/receipts` (upsert by channel + providerRef), `GET /v1/receipts?status=&from=&to=`
- `POST /v1/statement-imports` (multipart CSV + mapping profile), `GET /v1/statement-imports/{id}`
- `GET /v1/recon-runs?date=`, `POST /v1/recon-runs` (re-run a control for a date)
- `GET /v1/recon-exceptions?status=&type=&ageGt=`, `PATCH /v1/recon-exceptions/{id}` (assign, comment), `POST /v1/recon-exceptions/{id}/resolution`

## 7. Service-layer responsibilities

`ReceiptService`, `StatementImporter` (per-source mapping profiles), `Matcher`, `ControlRunner` (controls a–g as pluggable checks), `ExceptionService`.

## 8. Dependencies

accounting (clearing accounts, journals), loan-transactions, fixed-deposits, cob-jobs (runs after COB), organisation (payment types), platform-core.

## 9. Integration requirements

- veend-tenant and Giro post receipt statuses (a small change in their webhook handlers: the staged `Webhooks` collection already holds this data, [V] D2.3 F3).
- Rails' settlement files are provided by finance ops, or fetched by API later.

## 10. Authorization & permissions

`CREATE_RECEIPT` (service clients), `IMPORT_STATEMENT`, `READ_RECON`, `RESOLVE_RECONEXCEPTION` (+ `_CHECKER` above a threshold), `CREATE_CORRECTIVEJOURNAL` (+ `_CHECKER`, mandatory).

## 11. Transactional & accounting implications

No postings except through resolution: linked commands or maker-checked corrective journals. Clearing-account balances are the bridge between rails and the ledger.

## 12. Error handling & edge cases

- A statement re-imported with the same file hash is rejected.
- Partial settlement (provider nets fees): matched with a `FEE_NETTING` difference. Resolved by posting the fee expense (Dr PROVIDER_FEES / Cr clearing) via a corrective journal.
- A receipt `SUCCESS` → later `REVERSED` by the provider: if already posted, raise exception `PROVIDER_REVERSAL`, resolved by a CBA reversal command.

## 13. Testing requirements

Matching unit tests (ref, tolerance); control tests with injected drift; end-to-end: a duplicate callback produces one posting and zero exceptions; a pending payment produces no balance change and a visible pending item (guide UAT rows).

## 14. Implementation status & outstanding gaps

Not implemented. [A] Rail statement formats and settlement lags per provider; exception SLAs and owners (finance ops).
