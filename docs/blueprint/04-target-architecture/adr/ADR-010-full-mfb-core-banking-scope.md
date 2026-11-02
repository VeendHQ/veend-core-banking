# ADR-010: Fineract-Mod covers the full microfinance-bank CBS feature set

- Status: **Accepted** (stakeholder decision, 2026-11-02)
- Deciders: Engineering leadership
- Supersedes: the MVP boundary in ADR-001 §Decision and README "Key decisions" #2 (lending parity + GL + fixed deposits)

## Context

- The recovery team's "VeendHQ Core Banking Application Blueprint" ([input review](../../09-input-reviews/recovery-team-2026-10.md)) describes the feature set of a licensed microfinance bank's CBS, written against the BankOne menus Pettysave MFB uses.
- That feature set covers: deposits (savings, current, fixed, commitment); mortgages and quick loans; accounting and posting management; fixed assets and amortization; teller, vault and branch; statements; the full report families (incl. CBN and credit bureau); bills payments, transfers and NIBSS; payroll; shareholders; cards; and recovery data.
- Stakeholder decision: **"the new Fineract is to have all the features stated in the recovery team's documents."**

## Decision

1. Fineract-Mod's target scope is a **complete MFB core banking system**. It must provide **every feature** named in the recovery-team documents, in addition to the lending, fixed-deposit and LendLot/Giro compatibility scope already specified.
2. The authoritative feature list is the **feature traceability register** ([03-gap-analysis/recovery-team-feature-register.md](../../03-gap-analysis/recovery-team-feature-register.md)). Each feature maps to a module spec and a roadmap phase.
3. **The money-movement boundary changes for bank tenants.** For licensed deposit-taking tenants (e.g. an MFB), Fineract-Mod **moves money**: NIBSS transfers, bills payments, teller cash and card settlement. For Veend lending tenants and Giro, the earlier boundary stays: channels move money, and the CBA books it. Tenant type (`LENDER`, `TREASURY`, `BANK`) selects the enabled modules.
4. The ADR-001 build decision stands. Fineract remains the specification and oracle where it has equivalents (savings, current, FD/RD, collateral, guarantors, rescheduling, teller/cashier, credit bureau, shares). Features Fineract lacks (NIBSS, bills, cards, payroll, fixed assets, CBN returns, recovery case management) are specified from the recovery-team guide and Nigerian regulatory sources.

## Consequences

- MVP grows from 18 to about 30 modules. The roadmap adds a **banking track** (deposits, teller/branch, posting management, payments/NIBSS, cards) and an **enterprise track** (fixed assets, payroll, shareholders, regulatory reporting), and the timeline extends accordingly (roadmap §1).
- New external dependencies:
  - NIBSS (NIP/NEFT/BVN) membership and connectivity through the bank tenant's sponsor.
  - Bills aggregators (Entrazact, PurePay [A]: identify products and contracts).
  - A card processor/switch.
  - Credit bureaus (CRC, FirstCentral, CreditRegistry).
  - The CBN and NDIC reporting portals.
- Higher regulatory exposure: CBN IT standards, NDIC deposit insurance, PCI-DSS for the card scope, and NDPA for data protection. Security and compliance gates are added to the roadmap.
- A third migration source: **BankOne** (Pettysave MFB) ([migration §13](../../07-migration-strategy.md#13-bankone-pettysave-mfb-migration)).
- [A] **Build-vs-buy risk.** Building every vendor-CBS feature in house is substantially more work than lending + GL. R-ARC-06 tracks it, and leadership should re-confirm after the P-D discovery sizing.
