# D7 Technical Risk Assessment

Likelihood (L) and Impact (I) are scored 1–5. Score = L × I. **≥ 15 = red**, 8–14 = amber, ≤ 7 = green.

## 1. Architectural risks

| ID | Risk | L | I | Score | Mitigation | Owner |
|---|---|---|---|---|---|---|
| R-ARC-01 | **The rebuild underestimates Fineract's hidden semantics** (edge cases in schedules, allocation, accrual) | 4 | 5 | **20** | ADR-009 golden fixtures from the pinned Fineract version; historical replay of every migrated loan (D6 §6); MVP scope limited to the configurations in production (P-D D2); parity gates G2/G3/G4 | Tech lead |
| R-ARC-02 | The modular monolith decays into coupling | 3 | 3 | 9 | Lint-enforced import boundaries; ledger independent of lending (ADR-004); architecture review per phase | Architect |
| R-ARC-03 | Framework (template) upgrades conflict with hardening changes | 3 | 2 | 6 | Fork the hardened framework inside fineract-mod; upstream optional features only | Tech lead |
| R-ARC-04 | Sequelize limits for complex SQL and locking | 3 | 3 | 9 | Raw SQL for posting, COB and reports (ADR-002); Kysely allowed in reporting | Tech lead |
| R-ARC-05 | Compat facade becomes permanent | 3 | 3 | 9 | Usage metrics; time-boxed retirement (ADR-007); M5 on the roadmap | PO |

## 2. Financial calculation risks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-FIN-01 | Kobo-level differences in schedules or allocation versus Fineract | 4 | 5 | **20** | Decimal-only arithmetic (lint); 100% golden parity gate; property tests; finance hand-checks |
| R-FIN-02 | Wrong rounding mode or precision vs production configuration | 3 | 4 | 12 | Read production `c_configuration` and product digits (P-D D1/D2); per-product rounding |
| R-FIN-03 | Backdated transaction replay produces wrong deltas | 3 | 5 | **15** | Permutation property tests; replay recompute in reconciliation; maker-checker beyond the backdate window |
| R-FIN-04 | Value-date correction (DF-01) changes arrears and penalties for existing borrowers after M2 | 3 | 3 | 9 | Communicate; apply from the cut-over date forward only; finance approval |
| R-FIN-05 | **Accounting policy mismatch**: ACCRUAL_UPFRONT recognises all interest at disbursement, which may not comply with IFRS 9 effective-interest requirements; unclear NPA and write-off treatment of unearned interest | 3 | 4 | 12 | Finance policy workshop (P-D D6); both rules supported; policy change applied per product at a period boundary |
| R-FIN-06 | Interest recalculation or other unsupported product features found in production | 2 | 4 | 8 | P-D D2 inventory; scope added before G2 if found |

## 3. Accounting inconsistencies

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-ACC-01 | Existing LendLot GL is already inconsistent with operational reality (local write-offs DF-04, orphan approvals DF-06, unposted Vida repayments DF-09, possible double posts DF-02) | **5** | 4 | **20** | Faithful load, then explicit approved corrections T1–T5 (D6 §5.4), applied to both systems before the parallel run |
| R-ACC-02 | Unbalanced historical JE groups in Fineract | 2 | 4 | 8 | Quarantine + finance approval; migration-suspense account |
| R-ACC-03 | Hard-coded GL ids differ between tenants (assumed identical by veend-tenant) | 3 | 4 | 12 | Map by role from `acc_product_mapping` per tenant, never by id |
| R-ACC-04 | No per-rail clearing today, so post-cut-over reconciliation surfaces historical unexplained cash differences | 4 | 3 | 12 | Clearing accounts only for postings after cut-over; historical differences stay in FUND_SOURCE |

## 4. Data migration risks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-MIG-01 | The production Fineract version or schema differs from the repo version | 4 | 3 | 12 | P-D D1; ETL written against production DDL |
| R-MIG-02 | Duplicate or dirty client data blocks the uniqueness constraints | 4 | 2 | 8 | Pre-scan; quarantine; constraints created after cleanup |
| R-MIG-03 | Id collisions or translation errors break veend-tenant references | 2 | 5 | 10 | Reuse Fineract ids as PKs (D6 §5.2); sequences above max; contract tests on stored ids |
| R-MIG-04 | Long migration window for large tenants | 3 | 3 | 9 | Pre-load history days before; final delta at cut-over; rehearsal per tenant |
| R-MIG-05 | `Cbaloanarrears` global collisions (DQ-3) confuse post-migration dashboards | 3 | 2 | 6 | Rebuild from tenant-scoped CBA data (M4) |

## 5. Security vulnerabilities

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-SEC-01 | Credentials leaked in `APICalls` logs (DS-01) are reused against the new CBA | 4 | 4 | **16** | Rotate all LendLot credentials at M1; new CBA uses OAuth2; compat Basic credentials are new and IP-allowlisted |
| R-SEC-02 | Default credentials in production config (DS-02) | 3 | 5 | **15** | Fail-fast boot without secrets; M1 removes defaults in veend-tenant |
| R-SEC-03 | Cross-tenant data access | 2 | 5 | 10 | Role + schema isolation (ADR-003); automated cross-tenant tests; pen test |
| R-SEC-04 | MCP or admin tooling bypasses controls (framework G8) | 3 | 5 | **15** | MCP off in production; any tool goes through the CommandBus; route-table auth test |
| R-SEC-05 | PII (BVN/NIN) exposure through logs or search tags (G12) | 3 | 4 | 12 | Encryption + hash; log redaction tests; remove `searchIndex` |
| R-SEC-06 | TLS verification disabled in the veend-tenant client (DS-03) allows MITM | 3 | 4 | 12 | M1; certificate pinning optional |
| R-SEC-07 | Webhook spoofing to veend-tenant | 2 | 4 | 8 | HMAC with timestamp; replay window; secret rotation |

## 6. Performance bottlenecks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-PRF-01 | Hot GL balance rows (FUND_SOURCE) serialise postings | 3 | 3 | 9 | Striped balances; benchmark in P3 |
| R-PRF-02 | COB exceeds its window at scale | 2 | 4 | 8 | Partitioning; per-loan transactions; perf test with 1M loans (G4) |
| R-PRF-03 | Replay of long transaction histories on backdated posts | 2 | 3 | 6 | Snapshot allocations per installment; cap the backdate window |
| R-PRF-04 | Payroll-day spikes (bulk Remita/IPPIS postings) | 3 | 3 | 9 | Batch API; queue-level rate shaping in veend-tenant; horizontal API scaling |

## 7. Multi-tenancy risks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-TEN-01 | Schema migrations fail on a subset of tenants during deploy | 3 | 3 | 9 | Pre-deploy job blocks the deploy; expand/contract migrations; per-tenant version tracking |
| R-TEN-02 | Noisy tenant degrades others (COB, bulk posts) | 3 | 3 | 9 | Per-tenant rate limits and COB concurrency caps |
| R-TEN-03 | Cross-tenant on-lending semantics are wrong in the interim (duplicate loans) | 4 | 3 | 12 | Migrate as-is with links; target-model ADR in P3; build in P7 |
| R-TEN-04 | A lender requires a physically separate DB | 2 | 3 | 6 | ADR-003 escape hatch (per-tenant DSN) |

## 8. Integration challenges

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-INT-01 | Unknown or undocumented compat usage paths (dynamic or legacy code in veend-tenant) | 3 | 4 | 12 | Recorded-traffic corpus from `APICalls`; facade usage metrics; shadow mode surfaces unknown routes as 404 alerts |
| R-INT-02 | The custom `ActiveLoansSummary` definition is unavailable or undocumented | 3 | 3 | 9 | P-D D3; if missing, reverse-engineer from outputs plus code comments ([V] `Dashboard.js:1726-1753`) |
| R-INT-03 | veend-tenant change packages (M1–M4) compete with product roadmap work | 4 | 3 | 12 | M0 needs no change; M1–M3 are small and independently valuable on LendLot; schedule with the veend-tenant team early |
| R-INT-04 | Derived idempotency suppresses a legitimate duplicate payment | 2 | 3 | 6 | 24 h window, note-sensitive; ops override (`force=true` with maker-checker); M2 removes the need |

## 9. Operational and deployment risks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-OPS-01 | Team ramp-up on banking domain semantics | 4 | 4 | **16** | Finance SME embedded; Fineract code walkthroughs (this blueprint); pair reviews on ledger code |
| R-OPS-02 | No finance capacity for sign-offs at gates | 3 | 4 | 12 | Schedule gate reviews at kickoff; policy workshop first |
| R-OPS-03 | Cut-over collides with payroll deduction cycles | 2 | 4 | 8 | Cut-over calendar avoids deduction windows [A] |
| R-OPS-04 | Inadequate backups or restore never tested | 2 | 5 | 10 | PITR + quarterly restore drill before the first cut-over |
| R-OPS-05 | **Licensing**: fineract-mod template is GPL-3.0 ([V] `package.json`); Apache 2.0 obligations if any Fineract file is copied | 2 | 4 | 8 | Legal review before any external distribution; NOTICE handling (D1.1 §12); internal SaaS use only until cleared |
| R-OPS-06 | Running two CBAs in parallel for months increases ops load | 4 | 2 | 8 | Time-boxed per tenant (30-day window); automation of reconciliation |

## 9a. Fixed-deposit (Giro) risks

| ID | Risk | L | I | Score | Mitigation |
|---|---|---|---|---|---|
| R-FD-01 | **The investor liability is unbooked today** (FD-01). The opening balance constructed at migration may not match what finance or investors expect | 4 | 5 | **20** | Exact per-account reconciliation against Giro's own formula (D6 §12.3); finance sign-off on the opening liability; investor statements generated from both systems during a parallel month |
| R-FD-02 | Changing interest arithmetic alters investor payouts (float/rounding, tenure `'2'`, 90-day quarters) | 3 | 4 | 12 | `LEGACY_GIRO_DAILY_ROUNDED` for in-flight deposits; new rules for new deposits only; communicate |
| R-FD-03 | **Balance-sheet ownership is unclear** (Giro/Veend treasury vs lender tenant) for deposits that fund on-lending | 4 | 4 | **16** | D9 decision before P5-D build; the posting design supports either owner |
| R-FD-04 | Deployment and on-lending loans are not linked in any ledger (FD-13). Exposure per investor cannot be traced | 3 | 4 | 12 | `deposit_deployment` in MVP; full participation accounting in P7 |
| R-FD-05 | Giro and veend-tenant cut-overs interact: the Eden loan sits on a lender tenant migrated separately | 3 | 3 | 9 | Sequence: migrate the lending-to-lender tenant before or with Giro FD; deployment refs use `legacy_fineract_id` so they survive either order |
| R-FD-06 | Secret defaults in Giro config (FD-11) and logged CBA credentials | 3 | 4 | 12 | Giro package G1; rotate credentials at cut-over |
| R-FD-07 | Scope creep into the Giro wallet ledger (V3 SQL balances, liens on Fineract) | 3 | 3 | 9 | Explicitly out of MVP; a stub serves wallet compat calls; separate decision (D9) |

## 10. Top risks (red) and owners

| Rank | ID | Summary |
|---|---|---|
| 1 | R-ARC-01 / R-FIN-01 | Semantic parity with Fineract: golden + replay gates |
| 2 | R-ACC-01 | The existing books are already inconsistent: explicit corrections with finance |
| 3 | R-SEC-01 / R-SEC-02 / R-SEC-04 | Credential hygiene and admin-surface bypass |
| 4 | R-OPS-01 | Domain ramp-up |
| 5 | R-FIN-03 | Backdated replay correctness |
| 6 | R-FD-01 / R-FD-03 | Investor deposit liability unbooked today; unclear balance-sheet owner |
