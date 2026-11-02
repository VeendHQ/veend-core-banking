# Module (future): Loan Participation, Sale and Inter-Tenant Funding (on-lending)

> Phase: design in P3, build in P7 · Priority: P1 (because on-lending is live today) · Status: Outline spec
> Fineract reference: `fineract-investor` (external asset owners, `LoanAccountOwnerTransferBusinessStep`; [V] D1.1 §2, §8)
> veend-tenant touchpoints: on-lending/prime: `processOnLendingLoan` (`jobs.js:8369`), `disburseOnlendingCyclicLoan`, `bookExternalPrimeLoan`, `processVidaOnLendingLoan`, plus `Loans.onlenderLoanId(s)`, `onlenderAccessToken(s)`, `primeLenderOriginalLoanId` ([V] D2.3 F5)

> **Linked to fixed deposits.** Giro investor deposits are the funding source of on-lending: FD → prime-onlend pool → veend-tenant `/onlending/disburse` → loan to "Eden" on the lending-to-lender tenant ([V] [D1.4 §3.2](../../01-assessment/bill-payment-portal.md#32-how-fixed-deposits-fund-on-lending)). MVP records each deployment ([fixed-deposits FR-FD-11](../fixed-deposits.md)). This module adds the full participation and settlement accounting.

## 1. Overview & business purpose
Today a borrower loan funded by another lender is booked **twice**: in the originating tenant's CBA and in the funder's CBA (as a hidden "onlending" product), with independent schedules and no ledger link ([V] F5). Fineract-Mod should represent this as **one loan asset plus a funding relationship**. Possible models:
- **Participation:** the funder owns X% of the cash flows.
- **Sale / assignment:** the funder owns 100%, the originator services it.
- **Inter-tenant loan:** the funder lends to the originator, which lends to the borrower.

## 2. Functional requirements
- FR-INV-01: a funding agreement between tenants (rate, share, servicing fee, tenor rules).
- FR-INV-02: allocate a borrower loan to a funder at disbursement (saga across two tenant schemas, `platform.inter_tenant_transfer`).
- FR-INV-03: split each repayment by agreement, with entries in both tenants' GLs.
- FR-INV-04: settlement statements between tenants.
- FR-INV-05: buy-back and transfer.

## 3. Features & sub-features
Agreements · allocations · repayment waterfall · settlement · buy-back · reporting.

## 4. Business rules & validations
The funder's share of principal equals the funded amount. Interest is split per agreement. The servicing fee is retained by the originator. Allocations are atomic via a saga with compensation.

## 5. Data model
`platform.funding_agreement`, `platform.inter_tenant_transfer(id, from_tenant, to_tenant, type, amount, status, saga_state)`, `loan_funding(loan_id, agreement_id, share_pct, funded_amount)`.

## 6. API endpoints
`/platform/funding-agreements`, `POST /v1/loans/{id}/funding`, `GET /v1/funding/settlements`.

## 7. Service-layer responsibilities
`InterTenantSaga` (two local transactions + outbox-driven compensation); `RepaymentWaterfall`.

## 8. Dependencies on other modules
platform-core (tenancy), loan-lifecycle, loan-transactions, accounting.

## 9. Integration requirements
Giro transfers between lender accounts remain in veend-tenant; the CBA records the settlement reference.

## 10. Authorization & permissions
`PLATFORM_ADMIN` for agreements; `ALLOCATE_FUNDING` for the service client.

## 11. Transactional & accounting implications
- Originator: Dr LOAN_PORTFOLIO (100%), Cr FUNDING_PAYABLE (funder share).
- Funder: Dr INVESTMENT_IN_LOANS, Cr FUND_SOURCE.
- On repayment: split per waterfall.

[A] Requires finance and legal decision on the economic substance (sale vs. financing).

## 12. Error handling & edge cases
Saga failure mid-way (compensate); funder insolvency; borrower write-off (loss sharing).

## 13. Testing requirements
Saga chaos tests (kill between steps); a two-tenant GL reconciliation property.

## 14. Implementation status & outstanding gaps
- **Migration interim:** on-lending loans keep the current duplicate representation, migrated as-is (both tenants), linked via `attributes.onlenderLoanRef`.
- A target-model decision is needed before P7. [A] Business, finance and legal.
