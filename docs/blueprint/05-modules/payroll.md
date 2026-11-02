# Module: Payroll

> Phase: **P7-E** · Priority: P2 · Status: Spec draft
> Scope origin: F-55 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.7 "connect approved payroll journals to the GL; separately track salaries, deductions, taxes, liabilities and bank payments"; TB staff-cost and payroll-liability GLs (salary, housing, transport, 13th month, PAYE payable, pension payable, NHF, NHIS, HMO) [V]
> Fineract reference: none

## 1. Overview & business purpose

This module runs the bank tenant's **own staff payroll**, not borrower payroll lending: earnings, deductions, statutory taxes and contributions, net pay to staff accounts, and the GL journal, with liabilities tracked until they are remitted.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-PRL-01 | Employee master: staff id, grade, branch, cost centre, bank/CASA account, TIN, PFA and RSA PIN, NHF/NHIS ids |
| FR-PRL-02 | Pay elements: earnings (basic, housing, transport, utility, meal, dressing, entertainment, leave, 13th month, bonus) and deductions (PAYE, pension employee 8%, NHF 2.5%, NHIS, HMO, staff-loan repayments, cooperative) [A current statutory rates]; employer contributions (pension 10%) |
| FR-PRL-03 | Payroll run per month: calculate → review → approve (maker-checker) → post the journal → pay net salaries (internal CASA credit or NIBSS bulk) → payslips |
| FR-PRL-04 | **PAYE computation** per the Nigerian personal income tax schedule (consolidated relief, bands) [A: current law] |
| FR-PRL-05 | **Staff loan deductions** link to the staff-loan product in the loan module (repayment posted per run) |
| FR-PRL-06 | Liability remittance: PAYE to the state IRS, pension to PFAs, NHF, NHIS. A payment clears the payable GL |
| FR-PRL-07 | Reports: payroll register, payslips, statutory schedules, cost by branch/cost centre |

## 3. Features & sub-features

Employee master · pay elements · runs · PAYE · pension/NHF/NHIS · staff-loan deductions · net pay · remittances · payslips and reports.

## 4. Business rules & validations

BR-PRL-01: one approved run per month per pay group. BR-PRL-02: net pay ≥ 0 (deductions capped, with arrears carried). BR-PRL-03: payroll data is restricted (HR/finance roles only); salary details are excluded from general audit views.

## 5. Data model

```sql
CREATE TABLE employee (id BIGSERIAL PRIMARY KEY, staff_id BIGINT REFERENCES staff(id), grade TEXT, office_id BIGINT, cost_centre TEXT,
  pay_account_id BIGINT, tin_enc BYTEA, pfa TEXT, rsa_pin_enc BYTEA, status TEXT);
CREATE TABLE pay_element (code TEXT PRIMARY KEY, type TEXT CHECK (type IN ('EARNING','DEDUCTION','EMPLOYER')), formula TEXT, gl_account_id BIGINT, taxable BOOLEAN);
CREATE TABLE employee_pay_element (employee_id BIGINT, element_code TEXT, amount NUMERIC(19,6), from_date DATE, to_date DATE);
CREATE TABLE payroll_run (id BIGSERIAL PRIMARY KEY, period DATE, pay_group TEXT, status TEXT, journal_entry_id BIGINT, UNIQUE (period, pay_group));
CREATE TABLE payroll_line (run_id BIGINT, employee_id BIGINT, element_code TEXT, amount NUMERIC(19,6), PRIMARY KEY (run_id, employee_id, element_code));
```

## 6. API endpoints

`GET/POST /v1/employees`, `GET/POST /v1/pay-elements`, `POST /v1/payroll-runs`, `POST /v1/payroll-runs/{id}/approve`, `POST /v1/payroll-runs/{id}/pay`, `GET /v1/payroll-runs/{id}/payslips`, `POST /v1/statutory-remittances`.

## 7. Service-layer responsibilities

`PayrollCalculator` (pure, with a versioned tax-table module), run commands, `NetPayDisbursement` (CASA credits or [payments](payments-transfers-bills.md) bulk).

## 8. Dependencies

organisation (staff, branches), accounting, savings-current-accounts, loan-transactions (staff loans), payments-transfers-bills.

## 9. Integration requirements

Optional HRIS import; PFA/IRS remittance files [A].

## 10. Authorization & permissions

`MANAGE_PAYROLL`, `APPROVE_PAYROLL` (`_CHECKER`), `VIEW_PAYROLL` (restricted).

## 11. Transactional & accounting implications

Run posting: Dr staff-cost expense GLs per element / Cr SALARIES_PAYABLE, PAYE_PAYABLE, PENSION_PAYABLE, NHF_PAYABLE, NHIS_PAYABLE, STAFF_LOAN (repayments via the loan module). Net pay: Dr SALARIES_PAYABLE / Cr SAVINGS_CONTROL or NIBSS settlement. Remittance: Dr *_PAYABLE / Cr bank.

## 12. Error handling & edge cases

Mid-month joiners and leavers (pro-rate); retroactive pay changes (arrears element); a reversed run is only allowed before payment.

## 13. Testing requirements

PAYE table tests against published examples; run-to-GL tie-out; idempotent payment.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Whether payroll is currently in BankOne or an external HR system; statutory rates in force.
