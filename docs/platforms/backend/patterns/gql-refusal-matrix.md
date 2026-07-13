# GQL Refusal Matrix (Generator and Subagent)

This matrix defines when the agent must refuse implementation until missing inputs are provided.

## Refusal Classes

## R1: Missing Principal Semantics

- **Trigger**: request requires role scope (`me`/owner-bound query) without principal contract.
- **Refuse message**: "Cannot implement scoped query without principal context contract."
- **Required input**: principal source and expected unauthorized behavior.

## R2: Ambiguous Pagination Contract

- **Trigger**: root-many requested without selected mode and deterministic order.
- **Refuse message**: "Cannot implement root-many without bounded pagination and stable sort."
- **Required input**: mode (`pointer/page/upOnly`) and order field.

## R3: Missing ORM Field Mapping

- **Trigger**: requested filter/sort/extra references fields not declared in ORM loading plan.
- **Refuse message**: "Cannot map bridge behavior to ORM without required attr selection plan."
- **Required input**: required attrs list or include path.

## R4: Union/Polymorphic Ambiguity

- **Trigger**: union branch logic requested without discriminator contract.
- **Refuse message**: "Cannot implement union routing without discriminator mapping."
- **Required input**: discriminator attrs and branch mapping table.

## R5: Cross-Role Ownership Conflict

- **Trigger**: bridge/schema ownership mismatch or unclear target role.
- **Refuse message**: "Cannot register bridge without explicit role schema ownership."
- **Required input**: target schema ident and bridge ownership.

## R6: Style Conflict with Ejtmaa

- **Trigger**: request asks for incompatible style imported from non-matching project.
- **Refuse message**: "Requested pattern conflicts with Ejtmaa-compatible constitution."
- **Required input**: acceptable Ejtmaa-compatible adaptation.

## R7: Security Boundary Violation

- **Trigger**: proposed implementation moves critical auth checks to client/resolver only.
- **Refuse message**: "Cannot accept resolver-only security for scoped data."
- **Required input**: server-side guard placement in bridge/requester/model layers.

## R8: Incomplete Contract Change

- **Trigger**: schema/bridge change requested without required accompanying updates (types/registration/checks).
- **Refuse message**: "Cannot finalize partial contract changes that break registration/type/mapping invariants."
- **Required input**: complete affected files and expected contract behavior.

## R9: Unsafe SQL Filter Construction

- **Trigger**: request requires dynamic SQL filter with interpolated user input.
- **Refuse message**: "Cannot implement unsafe SQL interpolation in bridge filters."
- **Required input**: parameterized filter strategy (operators or replacements/binds).

## R10: Actor-Bound One Query Without Scope

- **Trigger**: bridge uses `STATIC` root-one without explicit owner-scoped `where`.
- **Refuse message**: "Cannot implement actor-bound one-query without ownership scope."
- **Required input**: ownership constraint fields for one-query (`owner_id`, `owner_type`, or equivalent).
