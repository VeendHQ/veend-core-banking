# Module: Close of Business (COB) and Scheduled Jobs

> Phase: P4 · Priority: P0 · Status: Spec draft
> Fineract reference: `fineract-cob`, `fineract-provider/.../cob/loan/*` ([V] `CheckLoanRepaymentOverdueBusinessStep`, `UpdateLoanArrearsAgingBusinessStep`, `SetLoanDelinquencyTagsBusinessStep`, `ApplyChargeToOverdueLoansBusinessStep`, `AddPeriodicAccrualEntriesBusinessStep`), `LoanCOBPartitioner`, `InlineLoanCOBExecutorServiceImpl`, `DelinquencyBucket`/`DelinquencyRange` ([V] `fineract-core/.../delinquency/domain/`), `m_business_date`, `m_loan_arrears_aging`
> veend-tenant touchpoints: none in the CBA today. veend-tenant computes arrears and defaults locally (`sweepLoanDefaults`, `markLoanDefaulted`, `checkOverdueTenureLoans` from the Aging report) ([V] D2.3 F7). [A] Whether LendLot runs Fineract's COB jobs in production is unknown; check `job` and `job_run_history`.

## 1. Overview & business purpose

End-of-day processing that keeps every active loan's financial state correct as of the business date:
- detect due and overdue installments
- compute arrears aging and delinquency classification
- apply overdue penalties
- post periodic accruals
- verify invariants
- produce report snapshots

It also hosts the scheduler for other periodic jobs.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-COB-01 | Run COB per tenant for the COB date (business date − 1), triggered automatically after the business-date roll |
| FR-COB-02 | Partition active loans (status 300/700, and 601 for recovery tracking) into chunks. Process them in parallel with bounded concurrency. Resumable |
| FR-COB-03 | Per-loan steps (in order): due check → overdue/arrears aging → overdue penalties → periodic accrual → delinquency classification → NPA/interest-suspension → invariant check → stamp `last_cob_date` |
| FR-COB-04 | Tenant steps: report snapshots (`rpt_aging_detail`, `rpt_active_loans_summary`), trial-balance check, `cob.completed` event |
| FR-COB-05 | Inline COB for a single loan before any online command when it lags behind |
| FR-COB-06 | Operations: view run status, retry failed partitions, re-run a date (idempotent), pause COB per tenant |
| FR-COB-07 | Delinquency buckets per product. The default set is the **CBN MFB prudential bands**: Performing (0), Pass & Watch (1–30), Substandard (31–60), Doubtful (61–90), Lost (≥ 91), with an IFRS 9 stage per band (see [writeoff-provisioning BR-WO-04/04b](writeoff-provisioning.md#4-business-rules)). Reporting bands for aging and maturity profiles (1–30, 31–60, 61–90, 91–180, 181–360, > 360, as in CBN MMFBR 763) are separate from classification. Tag history per loan |
| FR-COB-09 | **Interest-in-suspense step** after classification: moves newly due interest on Substandard-or-worse loans to INTEREST_IN_SUSPENSE (BR-WO-04c), and reverses it on cure |
| FR-COB-08 | Emit events: `loan.installment_due`, `loan.overdue` (first day), `loan.delinquency_changed`, `loan.npa_changed` |

## 3. Features

| Feature | MVP? |
|---|---|
| Business-date roll per tenant timezone | Yes |
| Partitioned, resumable COB | Yes |
| Arrears aging + delinquency buckets | Yes: replaces the local `daysOverdue` and `isDefaulted` logic over time (M4) |
| Overdue penalties | Yes (enabled per product) |
| Periodic accrual | Yes (for ACCRUAL_PERIODIC products) |
| NPA flag + interest suspension | Yes (policy configurable; [A] CBN prudential guidelines: 90 days) |
| Invariant check | Yes |
| Deposit steps (accrual, payout due, maturity, funding-window expiry) | Yes (P5-D; see [fixed-deposits](fixed-deposits.md)) |
| Loan interest recalculation step | No |
| Capitalised income / buy-down amortisation | No |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-COB-01 | **Days past due** (DPD) = `cobDate − oldest_unpaid_installment.due_date`, where an installment is unpaid when its outstanding exceeds `graceOnArrearsAgeing` tolerance (the product's `arrearsTolerance` amount) |
| BR-COB-02 | **Overdue** when DPD > `graceOnArrearsAgeing` days. `loan.overdue` is emitted on transition only |
| BR-COB-03 | **Bucket** = the range containing DPD. On a change, write a `loan_delinquency_history` row and emit an event |
| BR-COB-04 | **NPA** when DPD ≥ `npaAfterDays` (default 90). For ACCRUAL_PERIODIC, stop accruing interest on NPA loans and track suspended interest off-balance. For ACCRUAL_UPFRONT, [A] finance must decide on reversal of accrued but unpaid interest on NPA (R-FIN-05) |
| BR-COB-05 | **Periodic accrual** (ACCRUAL_PERIODIC): for each installment, accrue `interest_due × elapsedDays/periodDays` up to the COB date, minus already accrued. Post an ACCRUAL transaction (type 10) with a unique key `(loan, cobDate, 'ACCRUAL')`. Fees and penalties accrue on their due date |
| BR-COB-06 | **Penalties** per [charges BR-CHG-05], unique per (loan_charge source, installment, occurrence) |
| BR-COB-07 | **Invariant:** `loan.*_outstanding = Σ installments(due − paid − waived − written_off)`, and `Σ journal_line(LOAN_PORTFOLIO role accounts, loan_id) = principal_outstanding`. A violation sets the loan `integrity_hold=true` (blocking further commands except reads) and raises a P1 alert |
| BR-COB-08 | COB for date D cannot start until COB for D−1 has SUCCEEDED for that tenant |
| BR-COB-09 | During a partition's processing, online commands on its loans wait up to 3 s for the lock, then return 503 `COB_IN_PROGRESS` with `Retry-After` |

## 5. Data model

```sql
CREATE TABLE cob_run (id BIGSERIAL PRIMARY KEY, cob_date DATE UNIQUE NOT NULL, status TEXT NOT NULL, started_at TIMESTAMPTZ, finished_at TIMESTAMPTZ,
  partitions_total INT, partitions_ok INT, partitions_failed INT, stats JSONB);
CREATE TABLE cob_partition (run_id BIGINT REFERENCES cob_run(id), idx INT, loan_id_from BIGINT, loan_id_to BIGINT,
  status TEXT NOT NULL, attempts INT DEFAULT 0, error JSONB, PRIMARY KEY (run_id, idx));
CREATE TABLE delinquency_bucket (id BIGSERIAL PRIMARY KEY, name TEXT UNIQUE);
CREATE TABLE delinquency_range (id BIGSERIAL PRIMARY KEY, bucket_id BIGINT REFERENCES delinquency_bucket(id), classification TEXT, min_days INT, max_days INT);
CREATE TABLE loan_delinquency (loan_id BIGINT PRIMARY KEY REFERENCES loan(id), dpd INT NOT NULL, range_id BIGINT, overdue_since DATE,
  overdue_principal NUMERIC(19,6), overdue_interest NUMERIC(19,6), overdue_fee NUMERIC(19,6), overdue_penalty NUMERIC(19,6),
  is_npa BOOLEAN DEFAULT false, npa_since DATE, as_of DATE NOT NULL);
CREATE TABLE loan_delinquency_history (id BIGSERIAL PRIMARY KEY, loan_id BIGINT, range_id BIGINT, from_date DATE, to_date DATE);
-- loan_product.terms.delinquencyBucketId references delinquency_bucket
```

## 6. API endpoints

- `GET /v1/cob/runs?from&to`, `GET /v1/cob/runs/{date}`
- `POST /v1/cob/runs/{date}/retry` (failed partitions)
- `POST /v1/cob/pause`, `/resume`
- `POST /v1/loans/{id}/catch-up` (inline COB)
- `GET/POST /v1/delinquency-buckets`
- `GET /v1/loans/{id}/delinquency`

## 7. Service-layer responsibilities

- `CobOrchestrator` (clock-side): creates the run and partitions, enqueues Bull jobs.
- `CobPartitionWorker`: per loan, runs `LoanCobPipeline` steps in **one transaction per loan**, so a failure only affects that loan.
- `DelinquencyClassifier`, `AccrualCalculator`, `PenaltyApplier`, `InvariantChecker`, `ReportSnapshotter`.

## 8. Dependencies

platform-core (business date), loan-lifecycle, loan-transactions, charges, schedule engine, accounting, reporting (snapshots), events.

## 9. Integration

Events listed in FR-COB-08 and `cob.completed`. veend-tenant can drive arrears notifications and recovery assignment from `loan.overdue` and `loan.delinquency_changed` (M4), replacing the Aging-report polling.

## 10. Authorization

`READ_COB`, `EXECUTE_COB` (retry, catch-up), `UPDATE_DELINQUENCY_BUCKET`, `PAUSE_COB` (PLATFORM_ADMIN or TENANT_ADMIN).

## 11. Transactional & accounting implications

| Step | Postings |
|---|---|
| Periodic accrual | Dr INTEREST_RECEIVABLE / Cr INTEREST_ON_LOANS (and fees/penalties similarly) |
| Penalty applied (accrual) | Dr PENALTIES_RECEIVABLE / Cr INCOME_FROM_PENALTIES |
| NPA interest suspension (periodic) | No accrual; suspended interest tracked in `loan.attributes.suspendedInterest` and memo account [A] |
| Provisioning | Monthly, not daily (see [writeoff-provisioning](writeoff-provisioning.md)) |

## 12. Edge cases

| Case | Behaviour |
|---|---|
| Worker crash mid-partition | Per-loan transactions mean finished loans keep their `last_cob_date`. A retry skips them (`WHERE last_cob_date < cobDate`) |
| Business date advanced several days (outage) | One COB run per missed date, sequentially. Inline COB covers loans touched in between |
| Backdated repayment after COB has run for later dates | The replay recomputes allocations. The next COB re-derives DPD and bucket from current state (DPD is not incremental) |
| A loan in integrity hold | Skipped by COB and listed in the run report |

## 13. Testing

- Time-travel tests: fixed product, simulate 400 days of COB with repayment patterns, and compare DPD, bucket, accrual and penalties with Fineract golden output.
- Resumability: kill a worker at random points; the end state is identical to a clean run.
- Performance: 1M synthetic loans, 30-minute target (overview §5).

## 14. Status & gaps

- Not implemented.
- [A] Whether production LendLot runs COB or accrual jobs; if not, current GL accruals depend only on ACCRUAL_UPFRONT at disbursement.
- [A] The arrears policy that veend-tenant's `sweepLoanDefaults` uses, so that CBA buckets match the business definition of "default".
