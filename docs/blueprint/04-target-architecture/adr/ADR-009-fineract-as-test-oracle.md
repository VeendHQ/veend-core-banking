# ADR-009: Use Apache Fineract as a test oracle (golden fixtures and parallel run)

- Status: Proposed [R]

## Context

- Re-implementing schedules, allocation, accrual and accounting risks kobo-level divergence from what borrowers were promised and what the books show (R-FIN-01).
- Fineract ships extensive Cucumber scenarios ([V] `fineract-e2e-tests-runner/src/test/resources/features/`). LendLot holds years of real loans.

## Decision

1. **Golden fixtures.** A harness (built in P2) runs a local Fineract (docker-compose from the `fineract` repo, pinned to the LendLot production version [A]) with the tenant's **actual product configurations**.
   - It submits a generated matrix of inputs: principal, tenure, rate, charges, dates crossing month ends and leap years, and the repayment patterns early / partial / late / over / reversal.
   - It records `calculateLoanSchedule`, loan state after each transaction, and journal entries as JSON fixtures, committed to `test/golden/`.
2. **Historical replay.** For migrated loans, the full transaction history is replayed through the Fineract-Mod engine, and the result must equal Fineract's stored schedule and balances (D6 reconciliation).
3. **Parallel run.** During migration, selected tenants post to both systems (shadow mode) and nightly diffs must be zero (D6).

## Consequences

- We need a reproducible Fineract environment in CI or nightly (heavy). The fixtures are therefore generated offline and committed, and CI only replays them.
- Where Fineract behaviour is judged *wrong* for Veend, the deviation is documented as an intentional difference (`docs/blueprint/…/intentional-differences.md`, created when the first case arises) with finance sign-off.
