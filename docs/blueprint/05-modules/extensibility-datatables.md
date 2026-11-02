# Module: Extensibility (typed attributes, replacing datatables)

> Phase: P1 · Priority: P1 · Status: Spec draft
> Fineract reference: `DatatablesApiResource`, `x_registered_table`, runtime DDL
> veend-tenant touchpoints: datatable `userdata.email` ([V] `createBorrower`, `updateBorrower`) and datatable `loandata.{loanReason, channel, remitaMandateReference}` ([V] `createLoan`, `updateLoanDatatable`, 12 call sites)

## 1. Overview & business purpose

This module lets tenants attach custom fields to clients, loans and products **without runtime DDL**. It replaces Fineract datatables, whose runtime-created tables are hard to version, migrate and validate.

## 2. Functional requirements

| ID | Requirement |
|---|---|
| FR-EXT-01 | Define an attribute schema per entity type (`client`, `loan`, `loan_product`) as JSON Schema (draft 2020-12), versioned |
| FR-EXT-02 | Validate the `attributes` JSONB on write against the active schema version |
| FR-EXT-03 | Allow indexed search on attributes flagged `searchable` (GIN or expression index created by migration, not at runtime) |
| FR-EXT-04 | Compat: `PUT /datatables/{loandata\|userdata}/{id}?genericResultSet=true` maps to an attributes merge. Datatables arrays inside `POST /clients` and `POST /loans` map the same way |
| FR-EXT-05 | Seed the `loan` schema with `loanReason`, `channel`, `remitaMandateReference`, and the `client` schema with `email` (promoted to a column) |

## 3. Features

| Feature | MVP? |
|---|---|
| Per-entity JSON Schema, versioning | Yes |
| Compat datatable endpoints for `loandata`/`userdata` | Yes |
| Multi-row datatables (one-to-many) | No: model them as proper tables if needed |
| UI form generation | No |

## 4. Business rules

- **BR-EXT-01.** Unknown attribute keys are rejected on the native API. On the compat API they are stored under `attributes._legacy` with a warning log, so that veend-tenant is not broken.
- **BR-EXT-02.** Schema changes are additive within a version. Removing a field needs a migration that copies the data out.
- **BR-EXT-03.** Attribute values are **never** used in financial calculations.

## 5. Data model

```sql
CREATE TABLE attribute_schema (entity TEXT, version INT, schema JSONB NOT NULL, active BOOLEAN NOT NULL,
  PRIMARY KEY (entity, version));
-- entity tables carry `attributes JSONB NOT NULL DEFAULT '{}'`
CREATE INDEX ix_loan_attr_mandate ON loan ((attributes->>'remitaMandateReference'));
```

## 6. API endpoints

- `GET/POST /v1/attribute-schemas/{entity}`
- `PATCH /v1/{entity}/{id}/attributes` (JSON merge patch)
- Compat: `PUT /datatables/loandata/{loanId}`, `PUT /datatables/userdata/{clientId}`, `GET /datatables/userdata/{clientId}?genericResultSet=true` (shape `{columnHeaders:[{columnName}], data:[{row:[…]}]}`).

## 7. Service-layer responsibilities

`AttributeValidator` (ajv, compiled and cached per schema version), `AttributeMerger`.

## 8. Dependencies

platform-core. Consumed by clients, loan-lifecycle and loan-products.

## 9. Integration

Attribute changes are included in `client.updated` and `loan.updated` events.

## 10. Authorization

`UPDATE_<ENTITY>` covers attributes. `UPDATE_ATTRIBUTESCHEMA` is required to change a schema.

## 11. Transactional & accounting implications

None.

## 12. Edge cases

- `remitaMandateReference` is also stored on veend-tenant `Loans` ([V] DQ-5). The CBA copy is informational only.
- Migration: Fineract datatable rows are read via SQL from the registered tables and converted to JSONB.

## 13. Testing

Schema validation unit tests; compat datatable contract tests.

## 14. Status & gaps

- Not implemented.
- [A] Full production list of registered datatables and columns (`SELECT * FROM x_registered_table`).
