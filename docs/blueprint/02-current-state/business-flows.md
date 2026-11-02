# D2.3 Critical Business Flows (current state)

Sequence diagrams of the flows that touch the CBA. Function names and line references are from `veend-tenant` ([V] unless tagged).

## F1. Eligibility → application → approval (app channel)

```mermaid
sequenceDiagram
    autonumber
    participant B as Borrower app
    participant UC as Users.acceptTerms (Users.js:2081)
    participant J as jobs.getEligibility (jobs.js:360)
    participant LL as Lendlot client
    participant F as Fineract
    participant M as Mongo (tenant)
    Note over J: Earlier: eligibility job computes offer locally<br/>(salary x DTI, Remita/IPPIS history, decision engine)
    J->>LL: createBorrower (search by phone first)
    LL->>F: GET /search, POST /clients (+userdata.email)
    J->>LL: calculateLoan(product, amount, tenure)
    LL->>F: GET /loans/template, POST /loans?command=calculateLoanSchedule
    J->>M: Eligibilitys.offer {amount, tenure, interestRate, loanProduct, repayment}
    B->>UC: accept terms (eligibility id, amount, tenure)
    UC->>UC: top-up gate if an active loan exists<br/>(>=50% repaid OR Remita collection rate >= threshold)
    UC->>LL: createLoan(loanBody) [RequestId = elig._id-Date.now()]
    LL->>F: GET /loans/template, POST /loans (+loandata: loanReason, channel)
    UC->>LL: approveLoan(loanId, amount)
    LL->>F: POST /loans/{id}?command=approve (approvedOnDate=today)
    UC->>LL: getLoan(loanId, schedule only)
    LL->>F: GET /loans/{id}?associations=repaymentSchedule
    UC->>M: Loans.create {loanId, amount=totalPrincipalDisbursed,<br/>fee=feeChargesAtDisbursementCharged,<br/>totalRepayment=fee (fee treated as paid),<br/>totalOutstanding=totalOutstanding-fee,<br/>disbursementAmount=principal-fee, status 'approved'}
    UC->>M: Users.loans.push
    UC-->>B: loan approved, disbursement queued
```

Observations:
- [V] Application and approval happen in one synchronous step: no human approval, no maker-checker.
- [V] The **upfront fee is netted from the disbursement** (`disbursementAmount = totalPrincipalDisbursed - feeChargesAtDisbursementCharged`). Locally it is counted as already repaid. In Fineract it is a disbursement-time charge.
- [V] **Top-ups are not real top-ups.** `acceptTerms` marks the local loan `isTopup` but creates a second, independent Fineract loan. `isTopup/loanIdToClose` is commented out (`Users.js:2191-2196`), and so is the `loanToClose` link (`:2272`). `Users.disburseLoan` (`:2373-2387`) has net-off logic (`disbursementAmount -= prevLoan.totalOutstanding`), but it only runs when `loanToClose` is set, and nothing sets it today. In effect the borrower ends up with **two concurrent loans**. [A] Operations should confirm this is the intended behaviour (see DF-07).
- USSD variants (`Webhooks.ussd`, `shortUssd`, `shortVerificationUssd`, `shortEmployerVerificationUssd`, `shortUssdDropoff`) repeat this sequence inline.

## F2. Disbursement, requery and failure

```mermaid
sequenceDiagram
    autonumber
    participant Q as queue
    participant JD as jobs.disburseLoan (jobs.js:2233)
    participant RAIL as Giro / Remita
    participant CK as clock: requeryDisbursement (jobs.js:3579)
    participant PS as jobs.processDisbursementStatus (jobs.js:2996)
    participant LL as Lendlot
    participant F as Fineract
    participant M as Mongo
    Q->>JD: disburseLoan {loan}
    JD->>RAIL: transfer disbursementAmount to borrower account
    JD->>M: Loans.process='disbursed', requery=false, disbursementChannel
    loop every clock tick
        CK->>M: find Loans{requery:false, process:'disbursed', status != closed}
        CK->>Q: processDisbursementStatus(loan)
    end
    Q->>PS: processDisbursementStatus
    PS->>RAIL: paymentStatus / disburseStatus
    alt transfer succeeded
        PS->>LL: getLoan(loanId)
        PS->>RAIL: create Remita mandate from Fineract schedule<br/>(amount = totalRepaymentExpected - upfront fee)
        PS->>LL: updateLoanDatatable {remitaMandateReference}
        opt Giro path and Fineract still pendingApproval
            PS->>LL: approveLoan
        end
        PS->>LL: disburseLoan(loanId)
        LL->>F: POST ?command=disburse (actualDisbursementDate = today)
        PS->>M: status 'active', requery=true
    else failed, requeryCount >= 3
        PS->>LL: undoApproval + rejectLoan (errors swallowed)
        PS->>M: status 'closed', requery=true, reverse referral payout
    else pending
        PS->>M: requeryCount++
    end
```

Observations:
- [V] The Fineract `actualDisbursementDate` is the date the **requery confirmed** the transfer, not the date the money moved. If the requery runs late, interest accrual and due dates shift.
- [V] There is no `undodisbursal` path. If a transfer is reversed after Fineract disbursement, nothing corrects the CBA.
- [V] `undoApproval`/`rejectLoan` swallow errors (`{isError:true}`). A failed reject leaves an *approved* loan in Fineract while the local copy is `closed`.

## F3. Repayment: all channels funnel into `makeRepayment`

```mermaid
flowchart TB
    subgraph Sources
        PS[Paystack webhook] --> WH[(Webhooks collection)]
        FW[Flutterwave webhook] --> WH
        CP[CoralPay webhook] --> WH
        CG[Cgate USSD webhook] --> WH
        GIRO[Giro payment request /<br/>auto-debit webhook]
        MAN[Manual: GET /loans/payoff,<br/>multiplepayoff, bulkpayoff]
        REM[Remita payroll deductions]
        IPP[IPPIS payroll deductions]
        ONL[On-lending repay]
    end
    WH -- hourly: paystackRepayment,<br/>flutterwaveRepayment, coralPayRepayment,<br/>cgateUSSDRepayment --> PO[jobs.payoff jobs.js:3663]
    GIRO -- Webhooks.paymentrequestWebhook --> PO
    MAN --> PO
    REM -- clock: pushRepayments --> PR[jobs.processRepaymentFromRemita jobs.js:4508]
    IPP --> PI[jobs.processRepaymentFromIppis jobs.js:4864]
    PO -- unless paidOnLendlot --> MR[[lendlot.makeRepayment]]
    PR -- one call per new deduction --> MR
    PI --> MR
    ONL --> MR
    MR --> F[(Fineract POST transactions?command=repayment<br/>transactionDate = TODAY, paymentTypeId = 1)]
    MR -- if loan.loanDiskId --> LD[(LoanDisk)]
```

`jobs.payoff` detail ([V] `jobs.js:3663-3828`):

```mermaid
sequenceDiagram
    participant Q as queue
    participant P as jobs.payoff
    participant M as Mongo Loans
    participant LL as Lendlot
    participant F as Fineract
    Q->>P: {loanid, clientid, outstanding, purchaseReference?, paidOnLendlot?, note, tenantDb, config}
    P->>M: findOne {loanId}
    Note over P: reference = purchaseReference_DD-MM-YYYY<br/>OR eligibility000 + Date.now()  (non-deterministic)
    P->>P: totalOutstanding -= outstanding (float)<br/>totalRepayment += outstanding<br/>if totalExpected - totalPaid < 1: status = 'closed'
    alt not paidOnLendlot
        P->>LL: makeRepayment {loan, amount, employer}
        LL->>F: POST /loans/{id}/transactions?command=repayment
    end
    opt closed
        P->>Q: stopLoanCollection (cancel mandates)
        P->>P: referral and recovery commission via Giro
    end
    P->>M: loan.save()
```

Observations:
- [V] **Ordering hazard.** Local totals are computed first, then the CBA call is made, then `loan.save()` runs.
  - If the CBA call throws, nothing is saved locally. That is correct.
  - If `save()` fails after a successful CBA post, the job retries with a **new** `Date.now()` reference, so Fineract is posted **twice**.
- [V] Closure uses a **local** float comparison (`< 1` naira tolerance). Fineract decides closure independently.
- [V] Remita deductions are filtered to `paymentstatus === 'paid'` and **at least 2 days old**. Each is posted with RequestId `${loanId}_${deductiondate}`, which is deterministic and replay-safe, but `transactionDate` is still today.

## F4. Repayment reversal

```mermaid
sequenceDiagram
    participant A as Internal admin
    participant R as Loans.reverseRepayment (Loans.js:4549)
    participant LL as Lendlot
    participant F as Fineract
    participant M as Mongo
    A->>R: POST /loans/internal-admin/repayments/:id/reverse
    R->>M: RepaymentLogs.findById (reject source remita/ippis, require paid)
    R->>LL: getLoan(loanId, all)
    R->>R: find first transaction with type.code=repayment<br/>AND amount == log.amount AND !manuallyReversed
    R->>LL: undoRepayment(loanId, txId)
    LL->>F: POST /loans/{id}/transactions/{txId}?command=undo
    R->>M: Loans totals += / -=, reopen if closed
    R->>M: RepaymentLogs paymentStatus 'reversed' (guarded by paymentStatus:'paid')
    R->>R: queue logAudit
```

Observations:
- [V] The Fineract transaction is matched **by amount only**. Two equal repayments are ambiguous, so the wrong one may be reversed.
- [V] The "already reversed" guard runs **after** the Fineract undo. Two concurrent requests can both call undo (on different equal-amount transactions).

## F5. On-lending / prime (cross-tenant)

```mermaid
sequenceDiagram
    participant J as jobs.processOnLendingLoan (jobs.js:8369)
    participant AT as Accesstokens (global)
    participant LLp as Lendlot (funding tenant creds)
    participant Fp as Fineract tenant of funding lender
    participant G as Giro
    participant M as Mongo
    J->>AT: choose prime lender: preferred OR lastBalance > amount,<br/>tenure supported, round-robin by lastOnlendingLoan
    J->>LLp: createLoanProduct('onlending') if missing
    J->>LLp: createBorrower, createLoan, approveLoan
    J->>G: transfer from funding lender's Giro account
    J->>M: Loans.onlenderLoanId, onlenderAccessToken(s)
    Note over J: processOnlendingDisbursementStatus later disburses on Fp<br/>and copies the schedule into Loans.onLendingloanRepaymentSchedule
```

- [V] The same borrower loan therefore exists **twice**: in the originating tenant's CBA and in the funder's CBA, with independent schedules and no ledger link between them.
- [R] In Fineract-Mod this becomes an explicit **inter-tenant participation / loan-sale** construct. See [05-modules/future/investor.md](../05-modules/future/investor.md) and ADR-003.

## F6. Vida BNPL booking

- [V] Webhook → `logPendingVidaBnplLoansOnCba` (`jobs.js:16127`) claims the record atomically (`bookingStatus ∈ {pending, failed}` and `cbaLoanId: null`). It persists `cbaLoanId` **immediately after create**, before approve and disburse. `reconcileVidaBnplBooking` (`:16681`) completes partially booked loans ([V] `docs/vida/overview.md:99-105, 216-220`).
- [V] Vida repayments (`loan.repayment.success`) are acknowledged but **not posted to the CBA** ([V] `docs/vida/overview.md:308`).
- [V] Management fees are "recorded only, never charged on the CBA" (`overview.md:167`).

This flow is the best existing pattern for exactly-once booking. Fineract-Mod should make it the **default**, using server-side idempotency keys ([R]).

## F7. Write-off, default and insurance (local only)

- [V] `jobs.writeOffLoan` (`jobs.js:13548`) sets `isWriteOff`, `writeOffDate` and `writeOffBy` on the Mongo loan. **No CBA call.**
- [V] `sweepLoanDefaults` / `markLoanDefaulted` set `isDefaulted`, `daysOverdue` and `missedRepaymentCount` locally.
- [V] `jobs.insuranceClaimed` records the claim locally.
- Consequence: Fineract keeps the loan **active**, accruing interest under accrual accounting, and reports it in the portfolio. The GL never recognises the loss or the insurance recovery.

## F8. Product and charge administration

- [V] `Loanproducts.create` → `createLoanProduct` creates the product on the CBA with hard-coded accounting (ACCRUAL_UPFRONT, flat interest, fixed GL ids 5/9/30/32/31/24/6/10/7/8/19/33), then stores `externalId` locally.
- [V] `Lendlotcharges.create/updateOne/deleteOne` → charges CRUD. The local `lendlotChargeId` is stored.
- [V] The hard-coded GL ids only work if every tenant's chart of accounts was created with identical ids. [A] This presumably comes from the tenant-creator seed.
