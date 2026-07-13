# ORM Model Authoring Standard (Ejtmaa)

## Scope

This document is the operational constitution for creating or modifying ORM models in Ejtmaa.
Start from `docs/platforms/backend/patterns/orm-constitution-map.md` for execution order.

It governs:

- model shape,
- actor vs non-actor decisions,
- abilities and permissions,
- relations and ownership,
- ORM-to-GQL integration points,
- acceptance and rejection criteria.

## Hard Rules

1. Follow Ejtmaa style only.
2. Do not import structural style from non-matching projects.
3. Keep model security in model/requester layers, not resolver-only.
4. Keep list retrieval deterministic and bounded in all GQL-connected paths.
5. Keep every GraphQL-used field mapped to ORM attributes or bridge required attrs.

## Decision Tree

### Step 1: Is the model actor-sensitive?

Actor-sensitive means at least one:

- represents system identity owner directly,
- owns permissioned business actions,
- uses polymorphic owner semantics,
- exposes `can(...)` decisions for business operations.

If **yes**, follow Actor Path. If **no**, follow Non-Actor Path.

### Step 2: Does model participate in GraphQL query graph?

If **yes**:

- define bridge touchpoints,
- define required attrs for sort/filter/extras,
- define relation load expectations.

If **no**:

- still keep ORM contracts complete,
- no GQL mapping needed now, but keep fields future-safe.

## Actor Path (Required Pattern)

### 1) Type Contracts

- Define `type Attrs`.
- Define `type Ability`.
- Extend `Model<Attrs, CreationAttrs, Ability>`.

### 2) Structural Contracts

- `static attributes()` includes explicit owner and security-relevant fields.
- `static initOptions()` sets model/table names and required indexes/scopes.
- `static async boot()` defines owner and domain relations.

### 3) Ability Contracts

- Override `can<To extends keyof Ability, CanProps extends CanPropsBase<Ability[To]>>()`.
- Use `this.iCan(...)` for all branch execution.
- Each `Ability` branch must:
  - validate ownership/facts,
  - throw domain error keys for denial paths,
  - return success through `iCan`.

### 4) Owner Contracts

If polymorphic:

- use `owner_type` + `owner_id`,
- add scoped relations where applicable,
- keep `ownerIs(...)` and `getOwner(...)` utilities when used by requester logic.

## Non-Actor Path (Required Pattern)

### 1) Type Contracts

- Define `type Attrs`.
- Extend `Model<Attrs, CreationAttrs>`.
- Add `Ability` only when there is concrete domain need.

### 2) Structural Contracts

- Keep `attributes/initOptions/boot` complete.
- Prefer minimal scopes and minimal custom security logic.
- Keep helper functions pure and typed.

### 3) Security Contracts

- Authorization for write operations still belongs in requester/service orchestration.
- Do not move business security to resolver-only checks.

## Attributes Contract

Every model must satisfy:

1. `Attrs` includes persisted and used virtual fields.
2. `attributes()` reflects DB shape accurately.
3. For enum-like attrs, attach enum metadata consistently when needed by UI/GQL.
4. JSON payload fields are typed and marked appropriately.
5. Virtual getters must be deterministic and side-effect free.

### Enum-Like Attribute Contract (Required)

For enum-like persisted fields (example: `Supervisor.type`), use this exact pattern:

1. In `attributes()`:
   - use `DataTypes.STRING(191)` (project-default string width),
   - add enum metadata: `enum: ["one", "<enumKey>"]`,
   - add `isTypedObject: true`.
2. In model TypeScript contract:
   - derive the field type from translations using `G_Tr`,
   - preferred shape: `type XType = keyof G_Tr["enums"]["<enumKey>"]`.
3. If other files need that type:
   - export the type from the model and import it from there,
   - do not redefine duplicate enum unions in requesters.
4. Translation requirement:
   - the enum key must exist under both:
     - `backend/src/resources/trans/ar/general.ts`
     - `backend/src/resources/trans/en/general.ts`

This rule is aligned with Ejtmaa enum-like model metadata and localized enum mapping.

## Options Contract

Every model `initOptions()` must define:

- `modelName` and `tableName`,
- stable indexes for identity uniqueness and query performance,
- relevant scopes used by requester/bridge patterns.

Do not add scopes that are never consumed.

## Relations Contract

Every relation in `boot()` must document intent through naming and key direction:

- `belongsTo` for FK ownership,
- `hasOne/hasMany` for reverse traversal,
- polymorphic with `constraints: false` only when intentional and validated.

For GQL usage, relation direction must match bridge query access paths.

## Helper Functions Contract

Helpers inside model are allowed only when they:

- encapsulate domain behavior tied to the entity,
- are typed and deterministic,
- avoid accidental N+1 in caller flows,
- do not hide cross-boundary writes.

## ORM ↔ GQL Integration Contract

If model is queried through GQL:

1. Bridge must be able to load required attrs for:
   - root sorting,
   - root filtering,
   - extras,
   - relation resolution.
2. If bridge uses an attr not always selected, add it to `requiredOrmAttrs` or relation include attrs.
3. Any union/polymorphic resolution must have required discriminators loaded.
4. Keep `maxLevel`, `deepInclude`, and bounded list controls active.

## Requester Integration Contract

For write operations:

1. Validate facts in requester layer.
2. Call model `can(...)` for ability-protected operations.
3. Apply mutation after permission approval only.
4. Keep transaction and side effects deterministic.

## Anti-Patterns (Forbidden)

- Broad `any` usage for core model contracts.
- Resolver-only permission checks for mutations.
- Unscoped owner operations in actor models.
- Query logic depending on attrs not loaded by ORM selection.
- Blindly copying style from unrelated external repositories.

## Required Deliverables per New Model

For each model addition:

1. Model file with full contracts (`Attrs`, structure, relations, helper decisions).
2. Ability definition and `can()` branching decision (actor path only).
3. Bridge/schema integration notes (if GQL-facing).
4. Compliance checklist filled from `orm-model-compliance-checklists.md`.
5. Worked example mapping reference (actor/non-actor nearest example).

## Acceptance Criteria

A model change is accepted only when:

1. Style matches Ejtmaa baseline.
2. Actor/non-actor decision is explicit and justified.
3. Attr/options/relations/helpers are complete and typed.
4. Ability/can logic is complete for actor-sensitive operations.
5. GQL mapping is complete with no missing required attrs.
6. No forbidden anti-pattern is introduced.

## Related GQL Governance

When model changes affect GraphQL retrieval contracts, also apply:

- `docs/platforms/backend/patterns/gql-schema-bridge-authoring-standard.md`
- `docs/platforms/backend/patterns/gql-complex-problems-cookbook.md`
- `docs/platforms/backend/patterns/gql-anti-patterns-blacklist.md`
- `docs/platforms/backend/patterns/gql-refusal-matrix.md`
