# Module: Organisation (offices, staff, currencies, payment types, calendar)

> Phase: P1 · Priority: P1 · Status: Spec draft
> Fineract reference: `fineract-branch` (`m_office`, `m_staff`), `m_currency`, `m_organisation_currency`, `m_payment_type`, `m_holiday`, `m_working_days`
> veend-tenant touchpoints: `officeId: 1` hard-coded ([V] `lendlot/index.js:497`, `R_officeId=1`); `paymentTypeId: 1` ([V] `:728`); the office/staff client methods are dead ([V] D2.2 §5).

## 1. Overview & business purpose

This module holds the reference data that the lending and accounting modules depend on:
- the office hierarchy (for GL dimensions and data scope)
- staff (loan officers, recovery agents, if mastered in the CBA)
- enabled currencies
- **payment types**, which map collection rails to GL clearing accounts
- the calendar (holidays, working days) used by schedule due-date adjustment

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-ORG-01 | Seed a head office (id 1, `hierarchy='.'`) at tenant provisioning. Support child offices |
| FR-ORG-02 | CRUD for staff, with an `is_loan_officer` flag and office assignment |
| FR-ORG-03 | Enable currencies per tenant with `digits_after_decimal` and `in_multiples_of`. NGN is seeded |
| FR-ORG-04 | CRUD for payment types `{code, name, is_cash, clearing_gl_account_id, active}`. Seed: MANUAL, GIRO, REMITA, IPPIS, PAYSTACK, FLUTTERWAVE, CORALPAY, CGATE, ONEPIPE, VIDA, INSURANCE, MIGRATION |
| FR-ORG-05 | Define holidays per office (date range, reschedule-to date) and the tenant's working days (an RRULE `BYDAY`) |
| FR-ORG-06 | Provide a function to move a due date that lands on a non-working day or holiday, per product policy |
| FR-ORG-07 | **Branch management** for BANK tenants (recovery-team F-59): branch master (code, region, state/LGA, CBN branch code), branch status, branch GL segment, user-branch assignment. Operational controls are in [teller-branch-cash](teller-branch-cash.md) |

## 3. Features

| Feature | Sub-feature | MVP? |
|---|---|---|
| Offices | Head office seed, hierarchy, opening date | Yes (single office used by veend-tenant) |
| Staff | CRUD, loan-officer flag | Yes (minimal) |
| Currencies | NGN only enabled | Yes |
| Payment types | Rail-to-clearing-account mapping | **Yes, critical for reconciliation** |
| Holidays / working days | Due-date adjustment | Yes (product flag; default off to match current behaviour [A]) |

## 4. Business rules

| ID | Rule |
|---|---|
| BR-ORG-01 | An office's opening date is ≤ any client activation date or GL posting date in that office |
| BR-ORG-02 | An office's hierarchy path is `parent.hierarchy + id + '.'`, and is immutable after child records exist |
| BR-ORG-03 | Each payment type maps to exactly one active clearing GL account, of type ASSET |
| BR-ORG-04 | Holiday adjustment policy is one of `SAME_DAY`, `MOVE_TO_NEXT_WORKING_DAY`, `MOVE_TO_PREVIOUS_WORKING_DAY`, `MOVE_TO_NEXT_REPAYMENT_DAY` (Fineract `RepaymentRescheduleType`) |
| BR-ORG-05 | A currency cannot be disabled while open loans or non-zero GL balances use it |

## 5. Data model

```sql
CREATE TABLE office (id BIGSERIAL PRIMARY KEY, parent_id BIGINT REFERENCES office(id), hierarchy TEXT NOT NULL,
  name TEXT UNIQUE NOT NULL, external_id TEXT UNIQUE, opening_date DATE NOT NULL, legacy_fineract_id BIGINT UNIQUE);
CREATE TABLE staff (id BIGSERIAL PRIMARY KEY, office_id BIGINT NOT NULL REFERENCES office(id), firstname TEXT, lastname TEXT,
  mobile_no TEXT, email TEXT, is_loan_officer BOOLEAN DEFAULT false, active BOOLEAN DEFAULT true, joining_date DATE, external_id TEXT UNIQUE);
CREATE TABLE currency (code CHAR(3) PRIMARY KEY, name TEXT, digits_after_decimal SMALLINT NOT NULL, in_multiples_of SMALLINT, enabled BOOLEAN DEFAULT false);
CREATE TABLE payment_type (id BIGSERIAL PRIMARY KEY, code TEXT UNIQUE NOT NULL, name TEXT NOT NULL, is_cash BOOLEAN DEFAULT false,
  clearing_gl_account_id BIGINT REFERENCES gl_account(id), active BOOLEAN DEFAULT true, legacy_fineract_id BIGINT UNIQUE);
CREATE TABLE holiday (id BIGSERIAL PRIMARY KEY, name TEXT, from_date DATE, to_date DATE, repayments_rescheduled_to DATE, status TEXT);
CREATE TABLE holiday_office (holiday_id BIGINT REFERENCES holiday(id), office_id BIGINT REFERENCES office(id), PRIMARY KEY (holiday_id, office_id));
CREATE TABLE working_days (id SMALLINT PRIMARY KEY DEFAULT 1, recurrence TEXT NOT NULL DEFAULT 'FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR,SA,SU',
  repayment_reschedule_type SMALLINT NOT NULL DEFAULT 1, extend_term_daily_repayments BOOLEAN DEFAULT false);
```

## 6. API endpoints

- `GET/POST /v1/offices`, `GET/PATCH /v1/offices/{id}`
- `GET/POST /v1/staff`, `PATCH /v1/staff/{id}`
- `GET /v1/currencies`, `PUT /v1/currencies` (enable/disable)
- `GET/POST /v1/payment-types`, `PATCH /v1/payment-types/{id}`
- `GET/POST /v1/holidays`, `POST /v1/holidays/{id}/activation`
- `GET/PUT /v1/working-days`
- Compat: none. The dead methods `getEmployers`, `createStaff`, etc. are **not** served.

## 7. Service-layer responsibilities

- `CalendarService.adjustDueDate(date, officeId, policy)`: pure, cached per tenant.
- `PaymentTypeResolver.byCode(code)`: used by the transactions module.
- `OfficeScope.filter(principal)`: used by queries.

## 8. Dependencies

- Upstream: platform-core.
- Payment types reference `gl_account` (accounting). For seeding, payment types are created after the CoA template.
- Downstream: clients, loans, accounting, schedule engine (calendar).

## 9. Integration requirements

None external. [A] Public-holiday feed (Nigeria) could be imported yearly.

## 10. Authorization

`CREATE/UPDATE/READ_OFFICE`, `_STAFF`, `_PAYMENTTYPE`, `_HOLIDAY`, `UPDATE_WORKINGDAYS`, `UPDATE_CURRENCY`. Maker-checker is recommended on `UPDATE_PAYMENTTYPE`, because it changes GL routing.

## 11. Transactional & accounting implications

- Changing a payment type's clearing account affects **future** postings only. Historical entries are immutable.
- A change is a command with an audit trail.

## 12. Edge cases

- Holiday created after loans are disbursed: Fineract offers optional re-scheduling of affected installments. [R] For MVP, affect **new schedules only**, and make the option explicit later.
- Deleting a payment type used by transactions is prohibited. Deactivate it instead.

## 13. Testing

- Calendar adjustment unit tests per policy across month and year ends.
- Payment-type routing integration test (repayment via REMITA debits the REMITA clearing account).

## 14. Status & gaps

- Not implemented.
- [A] Confirm whether the live LendLot products use holiday or working-day rescheduling (`m_working_days` in production).
- [A] Confirm whether staff or loan officers should live in the CBA or stay in veend-tenant (recovery agents are veend-tenant concepts).
