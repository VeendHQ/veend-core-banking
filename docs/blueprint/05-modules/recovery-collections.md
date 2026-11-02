# Module: Recovery and Collections Case Management

> Phase: **P7-R** · Priority: P1 · Status: Spec draft
> Scope origin: F-61 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §8 (recovery data model: recovery officer, contact attempts, promises-to-pay, field visits, collateral, recovery status), §9 ("Recovery staff should not reduce principal outstanding merely because a customer promises to pay")
> Existing practice: veend-tenant recovery-agent assignment and commissions (`autoAssignOverdueLoansToRecoveryAgent`, `Loans.recoveryAgent*`, `Cbaloanarrears`; [V] D1.2 §3)
> Fineract reference: none (Fineract has delinquency tags only)

## 1. Overview & business purpose

Case management for delinquent loans: queues by bucket, assignment to recovery officers or agents, contact attempts, promises-to-pay (PTP) with tracking, field visits, escalation (legal, collateral realisation, guarantor call), and recovery performance reporting. Balances are **read-only** here. Money changes only through posted repayments, recoveries or approved restructures and write-offs.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-RCV-01 | Case creation automatically when a loan crosses a DPD threshold (on `loan.overdue` / `loan.delinquency_changed`), or manually |
| FR-RCV-02 | Assignment rules (bucket, branch, product, amount, round-robin, capacity) to recovery officers or external agencies; reassignment with history |
| FR-RCV-03 | Activities: call attempt (outcome code), SMS/email sent, field visit (geo, photo/notes), letter/demand notice; next action date |
| FR-RCV-04 | **Promise-to-pay**: amount, date. Auto-checked against posted repayments (kept/broken). A broken PTP escalates |
| FR-RCV-05 | Case view: the RT-04 balance definitions (principalPastDue, arrearsAmount, totalOutstanding, DPD, classification), contact data, collateral/guarantors, payment history, mandates |
| FR-RCV-06 | Escalations: legal referral, collateral realisation ([collateral-guarantors](collateral-guarantors.md) FR-COL-09), guarantor call, restructure proposal (→ [restructuring](restructuring-rescheduling.md) request) |
| FR-RCV-07 | Recovery status (`OPEN, CONTACTED, PTP, PARTIAL, CURED, ESCALATED, WRITTEN_OFF, RECOVERED, CLOSED`); auto-close on cure or closure |
| FR-RCV-08 | Commissions for agents/officers on recovered amounts (rules per agency), payable via payments. This replaces veend-tenant's ad-hoc commission logic over time |
| FR-RCV-09 | Reports and dashboards: collections by officer and bucket, PTP kept rate, roll rates between buckets, recovery after write-off |
| FR-RCV-10 | API for external recovery tools and analytics (read balances, write activities only) |

## 3. Features & sub-features

Cases · assignment · activities · PTP · case view · escalations · statuses · commissions · reports · external API.

## 4. Business rules & validations

BR-RCV-01: no endpoint here can change a loan balance. BR-RCV-02: a PTP's kept/broken status is evaluated at COB on the PTP date + grace. BR-RCV-03: contact rules follow the CBN consumer-protection and debt-recovery guidelines (contact hours, no harassment) [A]. BR-RCV-04: PII access is logged (FR-CLI PII audit).

## 5. Data model

```sql
CREATE TABLE recovery_case (id BIGSERIAL PRIMARY KEY, loan_id BIGINT UNIQUE NOT NULL, opened_on DATE, status TEXT, bucket TEXT, assignee TEXT, agency TEXT,
  next_action_on DATE, closed_on DATE, outcome TEXT);
CREATE TABLE recovery_activity (id BIGSERIAL PRIMARY KEY, case_id BIGINT, type TEXT, outcome TEXT, notes TEXT, geo POINT, at TIMESTAMPTZ, by_user TEXT);
CREATE TABLE promise_to_pay (id BIGSERIAL PRIMARY KEY, case_id BIGINT, amount NUMERIC(19,6), due_on DATE, status TEXT, matched_txn_ids BIGINT[]);
CREATE TABLE recovery_assignment_history (case_id BIGINT, assignee TEXT, from_ts TIMESTAMPTZ, to_ts TIMESTAMPTZ, reason TEXT);
CREATE TABLE recovery_commission (id BIGSERIAL PRIMARY KEY, case_id BIGINT, recovered_amount NUMERIC(19,6), commission NUMERIC(19,6), status TEXT, payment_id BIGINT);
```

## 6. API endpoints

`GET /v1/recovery-cases?bucket&assignee&status`, `POST /v1/recovery-cases`, `POST /v1/recovery-cases/{id}/assign`, `POST /v1/recovery-cases/{id}/activities`, `POST /v1/recovery-cases/{id}/ptp`, `POST /v1/recovery-cases/{id}/escalate`, `GET /v1/recovery-dashboard`.

## 7. Service-layer responsibilities

`CaseAutoCreator` (event consumer), `AssignmentEngine`, `PtpEvaluator` (COB step), `CommissionCalculator`.

## 8. Dependencies

loan-lifecycle (read model FR-LN-12/13), cob-jobs, collateral-guarantors, restructuring-rescheduling, writeoff-provisioning, payments-transfers-bills (commission payouts), events.

## 9. Integration requirements

The SMS/telephony provider (Termii is used by veend-tenant [V]); external agency portals via API; Metabase reads `rpt_views`.

## 10. Authorization & permissions

`READ_RECOVERYCASE`, `WORK_RECOVERYCASE` (activities, PTP), `ASSIGN_RECOVERYCASE`, `ESCALATE_RECOVERYCASE`, `APPROVE_COMMISSION` (`_CHECKER`).

## 11. Transactional & accounting implications

None, except commissions: Dr RECOVERY_COMMISSION_EXPENSE / Cr COMMISSION_PAYABLE, then paid via payments.

## 12. Error handling & edge cases

A loan cured and then re-delinquent reopens the same case with a new cycle; a written-off loan's case continues in "recovery after write-off" mode.

## 13. Testing requirements

PTP matching tests; assignment fairness; a guard test that no recovery endpoint can mutate balances (route-table test).

## 14. Implementation status & outstanding gaps

Not implemented. [A] Recovery team's current tooling and the fields it needs (the guide's §8 list adopted); agency commission rules.
