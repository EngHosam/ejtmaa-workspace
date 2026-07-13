# GQL Complex Problems Cookbook

This cookbook standardizes fixes for repeated high-risk GQL issues.

## Playbook 1: N+1 in Nested Relations

### Symptom

- slow query growth with nested relation selections,
- repeated DB calls per row in relation attr resolution.

### Root Cause

- relation paths not included in ORM plan,
- heavy `loadRelation` fallback on non-included paths.

### Canonical Fix

1. Keep relation loading through bridge include planning.
2. Keep `deepInclude` guards active.
3. Add required relation attrs when extras/conditions depend on them.
4. Restrict expensive extras to one-view (`ignoreExtra` for many).

### Verification

- inspect SQL count/shape for root-many and nested request,
- verify no per-row relation queries are introduced for common paths.

## Playbook 2: Pagination Drift (`pointer/page/upOnly`)

### Symptom

- inconsistent records between requests,
- duplicate/missing rows when switching pagination mode.

### Root Cause

- mixed pagination strategy in one bridge without deterministic order,
- mode-specific filters not applied consistently.

### Canonical Fix

1. Define one deterministic order for root-many.
2. Keep list clamp and offset policy in one helper (`withListable`).
3. Apply `upOnly` only when explicitly active.
4. Keep pagination mode contract documented in schema args/context.

### Verification

- compare page 1/2 snapshots with stable sort,
- compare upOnly incremental fetch against full list baseline.

## Playbook 3: Authorization Leakage at Root

### Symptom

- unauthorized actor can query role-scoped data.

### Root Cause

- missing principal check in `willPrepare` or `getRootOrmParent`.

### Canonical Fix

1. Guard principal presence before root query fetch.
2. Resolve owner-bound parent from context flags (`me`, role-specific markers).
3. Fail fast with stable `NOT_PERMIT` semantics.

### Verification

- run same query with and without principal context,
- unauthorized path must fail deterministically.

## Playbook 4: Polymorphic/Union Routing Mismatch

### Symptom

- wrong union branch selected,
- missing include branch data for polymorphic relation.

### Root Cause

- discriminator attrs not loaded,
- union include idents not mapped deterministically.

### Canonical Fix

1. Ensure discriminators are in required attrs.
2. Keep union mapping deterministic by explicit routing logic.
3. Align bridge relation config with schema union contract.
4. Re-check implementation against `docs/platforms/backend/patterns/gql-union-routing-reference.md`.

### Verification

- run mixed discriminator dataset query and validate branch correctness.
- verify union field resolves via bridge routing without schema-level union-member resolvers.

## Playbook 5: Missing Required ORM Attrs

### Symptom

- runtime undefined values in sort/filter/extras,
- bridge logic depends on non-selected attrs.

### Root Cause

- required attrs not declared in bridge required attr strategy.

### Canonical Fix

1. Add attrs to `requiredOrmAttrs` or include attrs.
2. Keep root-one expansion where extras require broad attr access.
3. Remove dead filter/extras references.

### Verification

- inspect generated SQL selected columns for the target query,
- validate all referenced attrs are selected.

## Playbook 6: Depth Explosion

### Symptom

- deep nested query causes high memory/time.

### Root Cause

- missing or relaxed depth guards.

### Canonical Fix

1. Keep finite `maxLevel`.
2. Keep strict `deepInclude` policy.
3. Limit relation auto-expansion in high-risk paths.

### Verification

- test deep nested query beyond allowed level and verify rejection/containment.

## Playbook 7: Expensive Extras on Lists

### Symptom

- list endpoints slow due to per-item extra computations.

### Root Cause

- extras computed for `many` responses without business need.

### Canonical Fix

1. Use `ignoreExtra` to block extras for non-one paths by default.
2. Allow extras on list only when explicitly justified.
3. Preload required attrs if list extras are unavoidable.

### Verification

- measure list query time with and without extras,
- ensure no expensive extra remains in default list path.

## Playbook 8: Context Contract Drift

### Symptom

- schema assumes context fields not always present.

### Root Cause

- divergent context builders across schemas/projects.

### Canonical Fix

1. Define schema context contract explicitly.
2. Ensure `by`, `lang`, list controls, and principal fields are always coherent.
3. Reject query if required principal missing for scoped paths.

### Verification

- test context variants (visitor/principal) and validate behavior matrix.

## Playbook 9: SQL Literal Injection Risk in Filters

### Symptom

- filter query builds `literal` string using user text,
- unexpected query errors or security risk.

### Root Cause

- interpolated user input passed into SQL fragments.

### Canonical Fix

1. Replace raw string interpolation with Sequelize operators where possible.
2. Use parameterized replacements/binds for dynamic values.
3. Restrict `literal` to constant fragments or validated numeric-only lists.

### Verification

- attempt malicious filter payload and ensure query remains safe,
- verify same business behavior with parameterized path.

## Playbook 10: `STATIC` Root-One Without Ownership Scope

### Symptom

- root-one id lookup can return records outside actor scope.

### Root Cause

- `getRootOrmParent` returns `STATIC`, but `getOrmFindOptions` misses owner-bound `where` constraints.

### Canonical Fix

1. If `STATIC` is used for actor-bound one-query, enforce owner scope in `where`.
2. Keep principal guard in `willPrepare` or `getRootOrmParent`.
3. Add regression checks for foreign-id access denial.

### Verification

- query id owned by current actor passes,
- query id owned by other actor fails.

## Playbook 11: Ability Matrix Extras Overload

### Symptom

- `abilities` extra triggers many `can()` calls and degrades one-view response.

### Root Cause

- large `loadExtra` fan-out with sub-args checks and repeated model capability evaluation.

### Canonical Fix

1. Keep heavy ability extras on `prepareType === "one"` only.
2. Load only asked ability nodes (selection-aware).
3. Ensure needed attrs are preloaded for ability branches.

### Verification

- measure one-view ability response with sparse vs dense asks,
- ensure list views do not execute ability fan-out.
