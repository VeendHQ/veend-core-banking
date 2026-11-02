# ADR-005: Typed command bus with idempotency, maker-checker and audit

- Status: Proposed [R]

## Context

- Fineract routes every write through `CommandWrapperBuilder` → `PortfolioCommandSourceWritePlatformServiceImpl` (maker-checker at `:80`) → `@CommandType` handlers. Each command is stored in `m_portfolio_command_source` ([V]).
- The newer `fineract-command` module moves to typed commands with hooks and a store ([V] D1.1).
- veend-tenant has no server-side idempotency guarantee, and its retries double-post ([V] DF-02).

## Decision

- **A single `CommandBus`.** Each command type declares:
  - a zod schema
  - its permission code
  - whether it is maker-checker eligible, plus an optional amount extractor for thresholds
  - its idempotency scope
  - a handler
- **Pipeline:** validate → idempotency lookup → authorize → (hold for approval) → transaction { lock, handle, post, outbox, audit, store response } → commit. See [processing §1](../processing.md#1-command-pipeline).
- **Idempotency** is keyed by `(tenant, principal.clientId, Idempotency-Key)` with a request-hash check. A failed command stores nothing, so retry is allowed.
- **Audit** is a `command_log` row per command (including rejected and failed ones, written in a separate autonomous transaction on failure) with a hash chain.
- **No bypass.** Repositories for financial tables are only exported to handlers (enforced by lint import boundaries). MCP and admin tools must call the bus.

## Consequences

- There is a little boilerplate per command, which is acceptable.
- Every write is replayable for audit and debugging from `command_log.payload_canonical`.
