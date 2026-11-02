# Module: Payments, Transfers, Bills and NIBSS

> Phase: **P6-C** · Priority: P0 for BANK tenants · Status: Spec draft
> Scope origin: F-47–F-54 ([register](../03-gap-analysis/recovery-team-feature-register.md)); guide §3.6, §6 (inbound repayment flow, payment lifecycle)
> Fineract reference: none for NIBSS/bills (Fineract has internal account transfers only, `AccountTransferDetails` [V])
> **Boundary change:** for BANK tenants Fineract-Mod moves money (ADR-010 §3). LENDER and TREASURY tenants keep the earlier boundary ([integrations-payments](integrations-payments.md)).

## 1. Overview & business purpose

Outbound and inbound interbank payments for a bank tenant:
- **NIBSS NIP** (instant) and **NEFT** transfers, with name enquiry;
- the **NIBSS institution directory**;
- **bills payment** (airtime, utilities, cable TV, etc.) through aggregators. The recovery team names **Entrazact** and **PurePay** [A: confirm the providers and contracts];
- **payment approvals**;
- the full **payment lifecycle** (instruction → validation → provider status → settlement → CBS posting → allocation → reconciliation), with pending/failed/reversed/disputed states always traceable.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-PAY-01 | **Institution management**: NIBSS institution list (bank/MFB codes, names, NIP and NEFT support, status), refreshed from NIBSS; manual overrides with audit |
| FR-PAY-02 | **Name enquiry** (NIP NE) before outbound transfer; the result is stored with the payment |
| FR-PAY-03 | **Outbound transfer**: debit CASA (or GL for corporate payments) → payment instruction → NIP FT → status (00 success / pending / failed) → TSQ (transaction status query) until final → on failure, auto-reversal of the debit. Fees and VAT. Limits (per txn, per day, per KYC tier) |
| FR-PAY-04 | **Inbound NIP credits**: receive the credit notification → validate the beneficiary NUBAN → post to CASA, or route to loan repayment/FD funding by mapping rules → acknowledge. Idempotent on the NIBSS session id |
| FR-PAY-05 | **Bills payment**: biller catalogue (category, biller, item, amount rules) per provider; validate the customer (e.g. meter/smartcard) → debit → vend → token/receipt → on failure, reversal |
| FR-PAY-06 | **Provider management**: adapters for NIBSS (via sponsor/aggregator), Entrazact, PurePay, plus a generic interface for future switches. Credentials in the secret manager. Per-provider health and failover |
| FR-PAY-07 | **Approvals**: corporate/bulk payments and above-limit transfers need maker-checker (unified inbox, FR-PST-05) |
| FR-PAY-08 | **Bulk payments**: file of beneficiaries → validation (name enquiry per row) → approval → execution → result file |
| FR-PAY-09 | **Lifecycle register**: every payment has states `INITIATED → VALIDATED → SENT → PENDING → SUCCESS / FAILED / REVERSED / DISPUTED`, with timestamps and provider responses. It feeds the [reconciliation](reconciliation-exceptions.md) receipt register |
| FR-PAY-10 | **Settlement reconciliation**: daily NIBSS settlement report and provider statements vs payments (auto-match by session id) |
| FR-PAY-11 | **Mandates / direct debit** (NIBSS e-mandate, Remita) for loan repayments: create, activate, debit, cancel. Linked to the loan schedule |
| FR-PAY-12 | Disputes: log, track and resolve customer payment disputes (shared with card disputes, [card-management](card-management.md)) |

## 3. Features & sub-features

Institution directory · name enquiry · NIP/NEFT outbound · inbound credits · TSQ and auto-reversal · bills (catalogue, validate, vend) · provider adapters (NIBSS, Entrazact, PurePay) · approvals · bulk payments · lifecycle register · settlement reconciliation · mandates · disputes.

## 4. Business rules & validations

| ID | Rule |
|---|---|
| BR-PAY-01 | **Debit-first with suspense**: the customer debit and the credit to NIP_OUTWARD_SUSPENSE are posted atomically before sending. A final failure reverses them, and success moves suspense → NIBSS settlement |
| BR-PAY-02 | A pending transfer is never shown as a completed payment (guide UAT). TSQ runs at 1, 5 and 30 min, then hourly up to the provider's window, then becomes an exception |
| BR-PAY-03 | Idempotency: client reference unique; NIBSS session id unique on inbound |
| BR-PAY-04 | The name-enquiry result must be < 15 min old at send time [A NIBSS rule] |
| BR-PAY-05 | Fees and VAT are posted with the payment. Stamp duty on qualifying inbound credits (≥ ₦10,000) [A current regulation] |
| BR-PAY-06 | Limits per KYC tier and channel (CBN) are enforced before debit |

## 5. Data model

```sql
CREATE TABLE financial_institution (code TEXT PRIMARY KEY, name TEXT, type TEXT, nip_enabled BOOLEAN, neft_enabled BOOLEAN, status TEXT, refreshed_at TIMESTAMPTZ);
CREATE TABLE payment (id BIGSERIAL PRIMARY KEY, client_ref TEXT UNIQUE NOT NULL, direction CHAR(1), rail TEXT, provider TEXT,
  debit_account_id BIGINT, beneficiary_bank TEXT, beneficiary_account TEXT, beneficiary_name TEXT, name_enquiry_ref TEXT,
  amount NUMERIC(19,6), fee NUMERIC(19,6), vat NUMERIC(19,6), narration TEXT, status TEXT, session_id TEXT UNIQUE,
  provider_response JSONB, status_history JSONB, approval_command_id BIGINT, created_at TIMESTAMPTZ DEFAULT now());
CREATE TABLE biller (id BIGSERIAL PRIMARY KEY, provider TEXT, category TEXT, name TEXT, code TEXT, items JSONB, active BOOLEAN);
CREATE TABLE bill_payment (payment_id BIGINT PRIMARY KEY REFERENCES payment(id), biller_id BIGINT, customer_ref TEXT, validation JSONB, token TEXT);
CREATE TABLE debit_mandate (id BIGSERIAL PRIMARY KEY, rail TEXT, mandate_ref TEXT UNIQUE, loan_id BIGINT, payer_account TEXT, payer_bank TEXT,
  max_amount NUMERIC(19,6), start_date DATE, end_date DATE, status TEXT);
CREATE TABLE payment_dispute (id BIGSERIAL PRIMARY KEY, payment_id BIGINT, type TEXT, status TEXT, opened_on DATE, resolution TEXT);
```

## 6. API endpoints

`GET /v1/institutions`, `POST /v1/name-enquiry`, `POST /v1/payments/transfers`, `POST /v1/payments/bulk`, `GET /v1/payments/{id}`, `POST /v1/payments/{id}/requery`, `GET /v1/billers?category`, `POST /v1/payments/bills/validate`, `POST /v1/payments/bills`, `POST /v1/mandates`, `POST /v1/mandates/{id}/debit`, `GET/POST /v1/payment-disputes`, `POST /internal/nibss/inbound` (provider callback, mTLS/IP-allowlisted).

## 7. Service-layer responsibilities

`PaymentOrchestrator` (state machine), `ProviderAdapter` interface (`nameEnquiry`, `transfer`, `tsq`, `billerCatalogue`, `validateBill`, `vend`) with `NibssNipAdapter`, `EntrazactAdapter`, `PurePayAdapter`; `InboundCreditRouter`; `TsqScheduler`; `SettlementReconciler`.

## 8. Dependencies

savings-current-accounts, loan-transactions, fixed-deposits, accounting, reconciliation-exceptions, platform-core (limits, approvals), clients (KYC tier).

## 9. Integration requirements

NIBSS NIP/NEFT/e-mandate (direct membership or a sponsor bank/aggregator [A]); Entrazact and PurePay APIs [A]; certificates and IP allowlisting; 24×7 availability for NIP inbound.

## 10. Authorization & permissions

`CREATE_TRANSFER` (limits), `APPROVE_TRANSFER` (`_CHECKER`), `CREATE_BILLPAYMENT`, `MANAGE_INSTITUTIONS`, `MANAGE_BILLERS`, `MANAGE_MANDATES`, `RESOLVE_DISPUTE`.

## 11. Transactional & accounting implications

| Event | Debit | Credit |
|---|---|---|
| Outbound initiated | SAVINGS_CONTROL (customer) | NIP_OUTWARD_SUSPENSE (amount), FEE_INCOME, VAT_PAYABLE |
| Outbound success (settled) | NIP_OUTWARD_SUSPENSE | NIBSS_SETTLEMENT (bank/sponsor) |
| Outbound failed | reversal of initiated | |
| Inbound credit | NIBSS_SETTLEMENT | SAVINGS_CONTROL / loan / FD |
| Bill payment | SAVINGS_CONTROL | BILLER_PAYABLE (provider); commission → INCOME_FROM_COMMISSIONS |

## 12. Error handling & edge cases

Timeout without a response → PENDING and TSQ (never auto-reverse on timeout alone). Duplicate inbound notification → ignored by session id. Success after a reversal (late success) → exception with a recall request. Provider outage → failover or queue with customer notice.

## 13. Testing requirements

NIBSS certification test pack [A]. Simulators for each provider. Chaos: timeouts and late successes. Reconciliation of 100% of payments daily.

## 14. Implementation status & outstanding gaps

Not implemented. [A] Who Entrazact and PurePay are (products, APIs, contracts); NIBSS connectivity model for the tenant; current Pettysave channel volumes.
