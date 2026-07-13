# ORM Constitution Map (Domain / Function / Expansion)

Use this map to navigate ORM governance without overlap.

## Domain Layer

- `orm-model-baseline-audit.md`
- `orm-model-authoring-standard.md`

## Function Layer

- `orm-model-worked-examples.md`
- `orm-model-compliance-checklists.md`

## Expansion Layer

- `models-owners-abilities-security.md`
- `gql-schema-bridge-authoring-standard.md` (when model changes affect GraphQL contracts)

## Execution Order (Mandatory)

1. Confirm model class category (actor / non-actor / polymorphic owner).
2. Apply authoring standard.
3. Validate checklist and worked example parity.
4. Validate owner/ability/security constraints.
5. Validate GraphQL integration impact if applicable.
