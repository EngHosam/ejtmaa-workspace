---
name: gql-union-routing-reference
description: Implements deterministic GraphQL union routing in Ejtmaa bridges using `static unions`, discriminator attrs, and `getUnionGQLIncludeIdent(...)`. Use when adding or fixing union/object polymorphic fields in backend GraphQL contracts.
---

# GQL Union Routing Reference

## When to Use

- Adding a GraphQL union field for model-backed data.
- Migrating separate relation fields into one union field.
- Fixing incorrect polymorphic branch selection.
- Auditing bridge/schema drift in union paths.

## Instructions

1. Define SDL union and field contract.
   - Add `union _X = _A | _B`.
   - Expose field as `x: _X`.
2. Keep concrete ORM relations (`a`, `b`) unchanged.
   - Do not invent fake ORM relation matching union field name.
3. In entity bridge:
   - define `static unions`,
   - include discriminator attrs in `requiredOrmAttrs`,
   - implement deterministic `getUnionGQLIncludeIdent(...)`.
4. Keep schema resolvers thin:
   - root bridge entry only,
   - no manual union-member field resolvers for model-backed paths.
5. Verify runtime behavior:
   - mixed discriminator dataset,
   - correct member type selection,
   - no cross-branch leakage.
6. Run verification and sync:
   - `yarn generate-types`,
   - `yarn type-check`,
   - sync backend GraphQL mirrors to the consuming frontends (website for customer; cpanel for supervisor).

## Core Runtime Notes

- `BridgeBase.prepareRelationsBridges()` prepares union member sub-bridges.
- `BridgeBase.loadRelations()` calls `getUnionGQLIncludeIdent(...)` to choose branch.
- Selected member attaches under union field key.
- Member payload `__typename` drives union type resolution.

## References

- `docs/platforms/backend/patterns/gql-union-routing-reference.md`
- `docs/platforms/backend/patterns/gql-schema-bridge-authoring-standard.md`
- `docs/platforms/backend/patterns/gql-complex-problems-cookbook.md`
- Ejtmaa pattern reference: `docs/platforms/backend/patterns/gql-union-routing-reference.md` (apply when a union field ships).
