# Module: <Name>

> Phase: P? · Priority: P0–P3 · Owner: TBD · Status: Spec draft
> Fineract reference: `<module/package>` · veend-tenant touchpoints: `<files>`

## 1. Overview & business purpose
What the module does, why it exists, and what it owns (its system-of-record boundaries).

## 2. Functional requirements
Numbered requirements in the form `FR-<MOD>-NN`, written so each can be tested.

## 3. Features & sub-features
| Feature | Sub-feature | MVP? | Notes |
|---|---|---|---|

## 4. Business rules & validations
Numbered rules in the form `BR-<MOD>-NN`. Include the exact formulas, rounding, date rules and state guards.

## 5. Data model & entity relationships
The tables and key columns (types, constraints, indexes), plus a Mermaid ER diagram.

## 6. API endpoints
The native `/v1` endpoints with request and response shapes, any LendLot-compat endpoints served, and their idempotency semantics.

## 7. Service-layer responsibilities
Commands, queries, domain services and the transaction boundaries.

## 8. Dependencies on other modules
Upstream modules this one needs, and downstream modules that consume it.

## 9. Integration requirements
Events emitted and consumed, and any external systems.

## 10. Authorization & permissions
The permission codes, and which commands are eligible for maker-checker.

## 11. Transactional & accounting implications
Journal postings (debit/credit by GL role), locking and consistency rules.

## 12. Error handling & edge cases
Error codes, plus the edge cases: concurrency, backdating, partial failure, reversal after close, and so on.

## 13. Testing requirements
Unit tests, property tests, golden fixtures against Fineract, integration tests and contract tests.

## 14. Implementation status & outstanding gaps
What exists today in fineract-mod (usually nothing), the open questions [A], and what is deferred.
