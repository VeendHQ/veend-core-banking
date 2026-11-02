# D2.5 Financial Transaction Lifecycle (current state)

This document shows how money and the records of it move today, and where the two can diverge.

## 1. Loan state: local vs. CBA

```mermaid
stateDiagram-v2
    direction LR
    state "veend-tenant Loans.status / process" as Local {
        [*] --> approved_local: acceptTerms creates doc (status approved, process approved)
        approved_local --> disbursed_pending: disburseLoan job (process disbursed, requery false)
        disbursed_pending --> active_local: requery OK (status active)
        disbursed_pending --> closed_failed: requeryCount >= 3 (status closed)
        active_local --> closed_local: payoff total paid within 1 naira of expected
        closed_local --> active_local: reverseRepayment reopens
        active_local --> active_local: isDefaulted / isWriteOff flags only
    }
    state "Fineract m_loan.loan_status_id" as CBA {
        [*] --> s100: POST /loans (100 submitted)
        s100 --> s200: approve (200)
        s200 --> s300: disburse (300 active)
        s200 --> s100: undoApproval
        s100 --> s500: reject (500)
        s300 --> s600: fully repaid (600)
        s300 --> s700: overpaid (700)
        s600 --> s300: undo repayment
    }
```

**Divergence points [V]:**

| Local state | CBA state | When it happens |
|---|---|---|
| `approved` | `200` | Normal, between acceptTerms and disbursement |
| `active` | `300` | Normal |
| `closed` (failed disbursement) | `200` or `100` | When `undoApproval` or `rejectLoan` failed silently (errors swallowed) |
| `closed` | `300` | Local closed within 1 naira tolerance while Fineract still shows a small balance (fees or rounding) |
| `active` + `isWriteOff` | `300`, accruing | Always after a write-off: never posted |
| `active` | `600`/`700` | Fineract closed by a payroll deduction before the local job synced |

## 2. Money flow vs. record flow

```mermaid
flowchart LR
    subgraph Real money
        FUND[Lender funding account<br/>Giro / Remita] -- disbursementAmount = principal - upfront fee --> BORROWER[Borrower bank account]
        BORROWER -- deductions / card / transfer --> COLLECT[Collection accounts<br/>Remita, Giro, Paystack...]
    end
    subgraph Records in Fineract
        D1[DISBURSEMENT tx: principal<br/>Dr Loan Portfolio / Cr Fund Source]
        D2[Upfront fee charge at disbursement<br/>Dr Fund Source / Cr Fee Income]
        R1[REPAYMENT tx: amount, date = today<br/>Dr Fund Source / Cr Portfolio, Interest Rcv, Fee Rcv]
        A1[Accrual upfront: interest and fees recognised<br/>Dr Receivable / Cr Income]
    end
    subgraph Records in veend-tenant
        L1[Loans.disbursementAmount, fee, totalRepayment = fee]
        L2[Loans.totalRepayment += amount<br/>totalOutstanding -= amount]
        L3[RepaymentLogs / Repayments / Webhooks]
    end
    FUND -.confirmed by requery.-> D1 & D2
    COLLECT -.payoff / payroll jobs.-> R1
    D1 -.-> L1
    R1 -.-> L2
    COLLECT -.-> L3
```

[V] The GL mapping above comes from `AccrualBasedAccountingProcessorForLoan` (see [D1.1 §5](../01-assessment/fineract.md#5-accounting)). The product's accounting rule is `ACCRUAL_UPFRONT` (4), from `createLoanProduct` in `lendlot/index.js:838`.

**Unreconciled gaps:**
1. **No settlement or suspense accounting.** Every repayment hits `FUND_SOURCE` (`paymentTypeId: 1`) whatever the rail. The GL cannot reconcile against Remita, Giro or Paystack settlement statements per channel. [R] Fineract-Mod should map payment types to per-rail clearing accounts.
2. **Value date.** Money is collected on day D, but the record is dated on the posting day (D+2 or later for Remita). Arrears, accrual and penalty calculations use the wrong date.
3. **No CBA record of failed or reversed transfers** after disbursement.
4. **Commissions** (referral, recovery) and **insurance** are paid through Giro with no GL entry in the CBA.
5. **Vida repayments** are not posted at all.

## 3. Transaction identity across systems

| Event | External reference | veend-tenant record | Fineract record | Link stored? |
|---|---|---|---|---|
| Disbursement | Giro/Remita transfer ref | `Loans.disbursementLog` | `m_loan_transaction` (type 1) | No |
| Repayment (card/USSD/transfer) | Paystack/Flutterwave/CoralPay ref; `purchaseReference` | `Webhooks`, `RepaymentLogs` | `m_loan_transaction` (type 2) | No (only an `APICalls` RequestId) |
| Repayment (payroll) | Remita/IPPIS deduction id + date | `Repayments.log`, `RepaymentLogs` | type 2 | Indirectly: RequestId `${loanId}_${deductiondate}` |
| Reversal | admin action | `RepaymentLogs.paymentStatus='reversed'` | `is_reversed=true` on the original | No |

[R] Fineract-Mod requires `external_ref` (unique per tenant + channel) on every financial transaction, returned in the response, so the link is stored on both sides. See [05-modules/loan-transactions.md](../05-modules/loan-transactions.md).
