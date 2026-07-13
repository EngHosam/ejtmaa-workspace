# GQL Complex Problems Playbooks Index

This index maps each high-risk problem class to the canonical remediation playbook.

## Covered Problem Classes

1. N+1 relation expansion
2. Pagination mode drift (`pointer/page/upOnly`)
3. Authorization boundary leaks
4. Polymorphic/union routing mismatch
5. Missing required ORM attrs
6. Depth/include explosion
7. Expensive extras on many responses
8. Context contract drift

## Source of Truth

Use:

- `docs/platforms/backend/patterns/gql-complex-problems-cookbook.md`

as the execution reference for:

- symptom detection,
- root-cause diagnosis,
- fix sequence,
- verification gates.
