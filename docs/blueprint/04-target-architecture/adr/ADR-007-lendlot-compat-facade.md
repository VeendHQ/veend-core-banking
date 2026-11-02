# ADR-007: LendLot-compatible API facade for zero-change cut-over

- Status: Proposed [R]

## Context

- veend-tenant reaches the CBA only through `services/lendlot/index.js`, using about 23 live methods, Fineract response shapes and positional report rows ([V] D2.2).
- Changing 85 instantiation sites and about 140 call sites in one release is risky ([V] grep counts).

## Decision

- Fineract-Mod exposes `/fineract-provider/api/v1/*` for **only** the live endpoints in D2.2 §2, with byte-compatible JSON shapes for the fields veend-tenant reads (D2.2 §3–4).
- The facade is a translation layer: compat request → native command → native result → Fineract-shaped response. No business logic.
- Cut-over per tenant becomes a **config change** in `Accesstokens.config.lendlotBaseUrl` (and credentials).
- The facade adds safety that Fineract lacks: derived idempotency, strict schemas for known-bad payloads, and real errors instead of silent failures where veend-tenant handles them. Any behaviour change is listed in D6 §4.
- The facade is time-boxed. veend-tenant migrates to a native `CbaClient` (M1–M5), and the facade is removed six months after the last tenant switches.

## Addendum: Giro as a second consumer

`bill-payment-portal` (Giro) uses its own fork of the client against a separate Fineract tenant. It relies on savings endpoints for fixed deposits, wallet mirror accounts and liens ([V] D2.7 §4). The facade covers those endpoints too. Cut-over for Giro is a change of `config.CBABaseURL` and credentials, scheduled on its own track (D6 §12).

## Consequences

- We must capture real LendLot request/response samples (`APICalls` collection) for contract tests.
- The custom `ActiveLoansSummary` report definition must be extracted from production (D2.2 §2.5 note).
