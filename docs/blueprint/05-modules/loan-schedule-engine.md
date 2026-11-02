# Module: Loan Schedule Engine

> Phase: P2 · Priority: **P0, highest financial risk** · Status: Spec draft
> Fineract reference: `LoanApplicationTerms`, `AbstractCumulativeLoanScheduleGenerator`, `CumulativeFlatInterestLoanScheduleGenerator`, `CumulativeDecliningBalanceInterestLoanScheduleGenerator`, `PaymentPeriodsInOneYearCalculator`, `ProgressiveEMICalculator` (future), `LoansApiResource:576-585` (`calculateLoanSchedule`)
> veend-tenant touchpoints: `calculateLoan` (eligibility repayment figure, USSD quotes), and every booked loan's figures copied from the schedule ([V] D2.2 L2, §4)

## 1. Overview & business purpose

A **pure, deterministic** library that turns (terms, principal, dates, charges) into an installment schedule: due dates, and principal, interest, fee and penalty due per installment, plus totals.

It is used by:
- the stateless calculate endpoint (quotes)
- loan submission and approval (the stored schedule)
- disbursement (regeneration from the actual disbursement date)
- COB (periodic accrual amounts)

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-SCH-01 | `generate(terms, principal, expectedDisbursementDate, charges, calendar)` returns `Schedule{periods[], totals}` |
| FR-SCH-02 | Interest methods: **FLAT** (MVP parity) and **DECLINING_BALANCE** with EQUAL_INSTALLMENTS (EMI) or EQUAL_PRINCIPAL |
| FR-SCH-03 | Frequencies DAYS/WEEKS/MONTHS with `repaymentEvery`. Optional first repayment date and interest-charged-from date |
| FR-SCH-04 | Graces: principal grace, interest-payment grace, interest-free periods |
| FR-SCH-05 | Charges per [charges BR-CHG-01..03] |
| FR-SCH-06 | Rounding per currency digits and product mode, plus `installmentAmountInMultiplesOf` |
| FR-SCH-07 | The output includes a "period 0" disbursement row, as Fineract does ([V] veend-tenant shifts it off: `periods.shift()` at `lendlot/index.js:96`) |
| FR-SCH-08 | Compat `POST /loans?command=calculateLoanSchedule` returns Fineract's shape: `currency, loanTermInDays, totalPrincipalDisbursed, totalPrincipalExpected, totalPrincipalPaid, totalInterestCharged, totalFeeChargesCharged, totalPenaltyChargesCharged, totalRepaymentExpected, totalOutstanding, periods[{period, fromDate[], dueDate[], daysInPeriod, principalDisbursed, principalLoanBalanceOutstanding, principalDue, interestDue, feeChargesDue, penaltyChargesDue, totalDueForPeriod, totalOutstandingForPeriod, totalOriginalDueForPeriod, …}]` |

## 3. Features

| Feature | MVP? |
|---|---|
| Flat, equal installments | **Yes (parity)** |
| Declining balance EMI, equal principal | Yes |
| Interest calculation period DAILY vs SAME_AS_REPAYMENT_PERIOD | Yes |
| Days in year/month variants | Yes |
| Holiday/working-day adjustment | Yes (policy flag) |
| Interest recalculation (reducing on prepayment) | No (P9) |
| Progressive (Fineract "progressive loan") | No (P9) |
| Variable installments, multi-tranche, down payment | No |

## 4. Business rules (algorithms)

**BR-SCH-01: Periods per year.** MONTHS: 12 / every. WEEKS: 52 / every. DAYS: `daysInYear / every` (Fineract `PaymentPeriodsInOneYearCalculator`).

**BR-SCH-02: Due dates.** `due[i] = start + i × every × unit`. Month arithmetic clamps to month end, e.g. Jan 31 + 1 month = Feb 28/29. Then apply the holiday policy. [A] Confirm the month-end clamping behaviour against golden fixtures.

**BR-SCH-03: FLAT interest** ([V] `LoanApplicationTerms.calculateTotalFlatInterestDueWithoutGrace` `:1137` and `calculateFlatInterestRateForLoanTerm`):

```
rateForTerm   = annualNominalRate / periodsPerYear / 100 × numberOfPeriodsInTerm
totalInterest = principal × rateForTerm                       (full precision, MathContext DECIMAL128 equivalent)
interest[i]   = round(totalInterest / n)  for i < n
interest[n]   = totalInterest − Σ interest[1..n-1]            (last installment absorbs the remainder)
principal[i]  = round(principal / n), last absorbs the remainder
```

where `annualNominalRate = interestRatePerPeriod × (12 if per MONTH)`. The **last-installment adjustment** matches `isLastRepaymentPeriod` handling at `LoanApplicationTerms.java:928-951` ([V]).

**BR-SCH-04: Declining balance, EMI.**

```
r   = periodic rate (annual / periodsPerYear / 100), or day-based when calculationPeriod = DAILY
EMI = P × r × (1+r)^n / ((1+r)^n − 1)   → rounded per currency, or to installmentInMultiplesOf
interest[i]  = outstanding[i-1] × r(i)   (r(i) day-weighted when DAILY: rateDaily × daysInPeriod)
principal[i] = EMI − interest[i];  last period principal = remaining outstanding
```

**BR-SCH-05: Charges.** Instalment fees are computed per period from that period's principal and/or interest (BR-CHG-01). Disbursement fees appear in period 0.

**BR-SCH-06: Rounding.** Use `Decimal` with precision 34 and round only at the per-component boundary. Mode per product (default HALF_EVEN). **Never round intermediate rates.**

**BR-SCH-07: Determinism.** The same inputs always give byte-identical output. No dependence on the clock, the locale or the timezone. Dates are plain `LocalDate`.

## 5. Data model

Pure functions, so there are no tables. Persisted output lives in `loan_installment` (owned by loan-lifecycle):

```sql
CREATE TABLE loan_installment (id BIGSERIAL PRIMARY KEY, loan_id BIGINT NOT NULL REFERENCES loan(id), number SMALLINT NOT NULL,
  from_date DATE NOT NULL, due_date DATE NOT NULL,
  principal_due NUMERIC(19,6) NOT NULL, interest_due NUMERIC(19,6) NOT NULL, fee_due NUMERIC(19,6) NOT NULL, penalty_due NUMERIC(19,6) NOT NULL DEFAULT 0,
  principal_paid NUMERIC(19,6) DEFAULT 0, interest_paid NUMERIC(19,6) DEFAULT 0, fee_paid NUMERIC(19,6) DEFAULT 0, penalty_paid NUMERIC(19,6) DEFAULT 0,
  interest_waived NUMERIC(19,6) DEFAULT 0, fee_waived NUMERIC(19,6) DEFAULT 0, penalty_waived NUMERIC(19,6) DEFAULT 0,
  principal_written_off NUMERIC(19,6) DEFAULT 0, interest_written_off NUMERIC(19,6) DEFAULT 0, fee_written_off NUMERIC(19,6) DEFAULT 0, penalty_written_off NUMERIC(19,6) DEFAULT 0,
  interest_accrued NUMERIC(19,6) DEFAULT 0, fee_accrued NUMERIC(19,6) DEFAULT 0, penalty_accrued NUMERIC(19,6) DEFAULT 0,
  completed_on DATE, obligations_met BOOLEAN GENERATED ALWAYS AS (completed_on IS NOT NULL) STORED,
  UNIQUE (loan_id, number));
```

## 6. API endpoints

- Native: `POST /v1/loan-schedules/calculate` (`{productId, principal, numberOfRepayments, interestRatePerPeriod?, expectedDisbursementDate, charges?}`). The response has amounts as strings.
- Compat: `POST /loans?command=calculateLoanSchedule` ([V] veend-tenant body fields listed in D2.2 L2).

## 7. Service-layer responsibilities

- `ScheduleGenerator` (interface) with `FlatScheduleGenerator`, `DecliningEmiScheduleGenerator`, `DecliningEqualPrincipalScheduleGenerator`.
- `PeriodsCalculator`, `DueDateGenerator`, `RoundingPolicy`.
- `CalculateScheduleQuery`: loads the product template and calls the generator. No DB writes.

## 8. Dependencies

charges (calculator), organisation (calendar), loan-products (terms). Consumed by loan-lifecycle, loan-transactions (replay needs the installment due amounts) and COB.

## 9. Integration

None.

## 10. Authorization

`CALCULATE_LOANSCHEDULE` (granted to SERVICE_LENDING).

## 11. Transactional & accounting implications

No postings. However, the totals become the receivables under ACCRUAL_UPFRONT at disbursement, so the schedule's correctness **is** GL correctness.

## 12. Edge cases

| Case | Expected |
|---|---|
| Principal not divisible by n | Last installment absorbs the remainder (BR-SCH-03) |
| Disbursement on the 29th–31st | Month-end clamping (BR-SCH-02); golden-test Jan 31, Feb 29 and Mar 31 starts |
| Leap year with daysInYear=ACTUAL | Day-based interest uses 366 for the leap-year portion as Fineract does [A golden] |
| Rate 0% | Interest 0; schedule still produced |
| Tenure 1 | Single installment, all principal and interest |
| Holiday on the last due date | Policy applies; maturity moves |
| `installmentAmountInMultiplesOf` | EMI rounded up to the multiple; the last installment adjusts |

## 13. Testing (gate for P2 exit)

1. **Golden fixtures (ADR-009).** At least 2,000 generated cases per live product configuration, plus all edge cases above, exported from a Fineract instance at the production version. 100% must be kobo-exact (every period component and every total).
2. **Property tests (fast-check).** Σ principal_due = principal; Σ interest_due = totalInterest; all components ≥ 0; due dates strictly increasing; determinism.
3. **Historical sample.** For 1,000 random migrated loans, regenerate from stored terms and compare with Fineract's stored `m_loan_repayment_schedule`. Any diff blocks P2 exit.
4. **Fineract Cucumber ports.** Translate the relevant `EMICalculation-*.feature` scenarios ([V] `fineract-e2e-tests-runner/src/test/resources/features/`).

## 14. Status & gaps

- Not implemented.
- [A] The production Fineract version (algorithms changed across 1.x releases; golden fixtures must use the same version).
- [A] Product settings: `daysInYearType`, `daysInMonthType`, `interestCalculationPeriodType`, `allowPartialPeriodInterestCalcualtion` (all taken from the template by veend-tenant; [V] `createLoanProduct` `:842-847`).
