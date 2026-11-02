# Module (future): Groups, Centers and Group Loans (JLG/GLIM)

> Phase: P7+ · Priority: P3 · Status: Outline spec
> Fineract reference: `m_group`, group/center resources, GLIM (`glimrepayment` command at [V] `LoansApiResource.java:694`)
> veend-tenant touchpoints: none. Lending is individual, payroll-backed ([V] D1.2 §3).

## 1. Overview & business purpose
Microfinance-style group lending: clients organised into groups and centers, with joint-liability or group loans. This is relevant only if a lender runs MFI-style products.

## 2. Functional requirements
- FR-GRP-01: group and center CRUD, with membership.
- FR-GRP-02: group loan (single account) and GLIM (parent with child loans).
- FR-GRP-03: group repayment allocated to members.
- FR-GRP-04: meeting calendar alignment of due dates.

## 3. Features & sub-features
Groups · centers · membership · group loans · GLIM · collection sheets.

## 4. Business rules & validations
A client may belong to N groups per policy. GLIM children share their terms. Group repayment is split pro rata to dues.

## 5. Data model
`client_group`, `group_member`, `center`, `glim_account` (parent of `loan`).

## 6. API endpoints
`/v1/groups`, `/v1/centers`, `/v1/groups/{id}/members`, `/v1/glim-accounts`, `/v1/collection-sheets`.

## 7. Service-layer responsibilities
Group commands; GLIM repayment splitter.

## 8. Dependencies on other modules
clients, loan-lifecycle, loan-transactions, organisation (calendar).

## 9. Integration requirements
None.

## 10. Authorization & permissions
`*_GROUP`, `*_CENTER`, `*_GLIMLOAN`.

## 11. Transactional & accounting implications
Same as individual loans per child; the parent carries no GL.

## 12. Error handling & edge cases
Member exit with outstanding loans; group dissolution.

## 13. Testing requirements
Fineract GLIM integration scenarios as golden fixtures.

## 14. Implementation status & outstanding gaps
Not planned. [A] Business need.
