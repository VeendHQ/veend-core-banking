# Module: Posting Management (manual, batch, transfers, approvals, reversals)

> Phase: **P6-B** · Priority: P0 for BANK tenants · Status: Spec draft
> Scope origin: F-09, F-10, F-12–F-15 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.1 "Posting Management includes account posting, batch uploads, cash posting, funds transfers, loan repayment posting, posting approvals and reversals"
> Fineract reference: `JournalEntriesApiResource` (manual JEs), `AccountTransfers*` ([V] `AccountTransferDetails`), batch API

## 1. Overview & business purpose

This module is the back-office posting desk. It covers:
- multi-leg postings across customer accounts and GL;
- batch uploads;
- internal funds transfers;
- manual loan-repayment posting;
- one approvals inbox;
- controlled reversals.

It is a **front end to existing commands**: every posting still goes through the command bus and the ledger rules. Cash posting is in [teller-branch-cash](teller-branch-cash.md).

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-PST-01 | **Account posting**: one balanced entry with N legs. A leg may target a GL account (manual-entries-allowed only), a CASA account, a loan (as repayment or charge), or an FD. The system validates the balance, limits and closures |
| FR-PST-02 | **Funds transfer** (internal): CASA → CASA / loan / FD, same or different client. Standing instructions are in [savings-current-accounts](savings-current-accounts.md) |
| FR-PST-03 | **Loan repayment posting**: manual single or bulk, with the payment type and external ref (routes to `MakeRepayment`) |
| FR-PST-04 | **Batch upload**: CSV/XLSX template per batch type (GL journal, CASA credits/debits, loan repayments, transfers). Pipeline: validate all rows → preview with totals and control sum → submit for approval → execute row by row (each idempotent) → result file. A batch is all-or-nothing by default, with an optional partial mode |
| FR-PST-05 | **Posting approvals inbox**: a unified view of maker-checker items (postings, batches, reversals, transfers, payments) with amount, maker, age and SLA |
| FR-PST-06 | **Reversals**: select a posting/transaction → reason → approval → exact contra posting linked to the original. Loan transactions use the replay engine ([loan-transactions](loan-transactions.md)) |
| FR-PST-07 | **Entry codes** (BankOne-style, e.g. C102/D102 [V call-over]): a configurable transaction-code catalogue that drives narration, GL behaviour and reporting groupings |
| FR-PST-08 | **Instrument numbers** (cheque/voucher), narration templates and attachments |

## 3. Features & sub-features

Account posting · internal transfers · loan repayment posting · batch uploads (templates, validation, approval, results) · approvals inbox · reversals · entry codes · attachments.

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-PST-01 | Σ debit = Σ credit per posting (the ledger constraint) |
| BR-PST-02 | GL legs are only allowed on accounts with `manual_entries_allowed`. Control accounts (loan portfolio, savings control, suspense for NIP) are **forbidden** for manual legs, because they change only through sub-ledger commands |
| BR-PST-03 | Batch control: declared row count and control total must match the parsed file. File hash unique |
| BR-PST-04 | Every manual posting and batch needs maker-checker above zero by default (tenant-configurable thresholds); maker ≠ checker |
| BR-PST-05 | A reversal must reference the original; a transaction cannot be reversed twice; reversal into a closed period follows BR-ACC-03 |
| BR-PST-06 | Limits per role (FR-PLT-11) apply per posting and per batch total |

## 5. Data model

```sql
CREATE TABLE manual_posting (id BIGSERIAL PRIMARY KEY, entry_code TEXT, value_date DATE, narration TEXT, instrument_no TEXT,
  status TEXT, command_id BIGINT, journal_entry_id BIGINT, attachments JSONB);
CREATE TABLE manual_posting_leg (posting_id BIGINT, seq SMALLINT, target_type TEXT CHECK (target_type IN ('GL','CASA','LOAN','FD')),
  target_id BIGINT, debit NUMERIC(19,6) DEFAULT 0, credit NUMERIC(19,6) DEFAULT 0, PRIMARY KEY (posting_id, seq));
CREATE TABLE posting_batch (id BIGSERIAL PRIMARY KEY, type TEXT, file_hash CHAR(64) UNIQUE, rows INT, control_total NUMERIC(19,6),
  mode TEXT, status TEXT, maker TEXT, checker TEXT, result_file TEXT);
CREATE TABLE posting_batch_row (batch_id BIGINT, row_no INT, payload JSONB, status TEXT, error TEXT, command_id BIGINT, PRIMARY KEY (batch_id, row_no));
CREATE TABLE entry_code (code TEXT PRIMARY KEY, description TEXT, direction CHAR(1), report_group TEXT, active BOOLEAN);
```

## 6. API endpoints

`POST /v1/postings`, `GET /v1/postings/{id}`, `POST /v1/transfers/internal`, `POST /v1/batches` (upload), `GET /v1/batches/{id}`, `POST /v1/batches/{id}/submit`, `GET /v1/batches/{id}/result`, `GET /v1/approvals?type&status`, `POST /v1/reversals` (`{targetType, targetId, reason}`), `GET/POST /v1/entry-codes`.

## 7. Service-layer responsibilities

`PostingDesk` (translates legs to commands), `BatchProcessor` (parse → validate → stage → execute with per-row idempotency `batchId:rowNo`), `ApprovalsInbox` (query over `command_log` PENDING_APPROVAL), `ReversalService`.

## 8. Dependencies

platform-core, accounting, savings-current-accounts, loan-transactions, fixed-deposits, teller-branch-cash.

## 9. Integration requirements

File templates documented and versioned. Optional SFTP drop for batch files [A].

## 10. Authorization & permissions

`CREATE_POSTING` (+ `_CHECKER`), `UPLOAD_BATCH` (+ `_CHECKER`), `CREATE_INTERNALTRANSFER`, `REVERSE_TRANSACTION` (+ `_CHECKER`), `MANAGE_ENTRYCODES`.

## 11. Transactional & accounting implications

Each posting is one journal entry. A batch executes as N commands, each with its own transaction. In all-or-nothing mode, the batch is validated fully first and execution runs inside one DB transaction up to a size limit [A: 5,000 rows].

## 12. Error handling & edge cases

Partial batch failure in partial mode leads to a result file with per-row errors and a re-submit of the failed rows only. Duplicate file → rejected. A posting to a dormant account follows the CASA rules.

## 13. Testing requirements

Batch validation fixtures (bad totals, bad accounts, duplicates); the guide UAT "duplicate upload → only one financial posting"; reversal linkage.

## 14. Implementation status & outstanding gaps

Not implemented. [A] BankOne entry-code catalogue at Pettysave (to migrate); batch volumes.
