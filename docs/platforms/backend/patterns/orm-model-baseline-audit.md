# ORM Model Baseline Audit (Ejtmaa)

## Purpose

This audit locks the canonical model authoring style for Ejtmaa and documents what is accepted, optional, and rejected for future model generation.

## Canonical References

- `backend/src/app/orm/models/Model.ts`
- `backend/src/app/orm/models/User.ts`
- `backend/src/app/orm/models/Notification.ts`
- `backend/src/app/orm/models/Customer.ts`
- `backend/src/app/orm/models/Supervisor.ts`
- `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`
- `backend/src/app/gql/schemas/CustomerSchema.ts`

## Baseline Invariants (Must Keep)

1. **Model Base Contract**
   - All app models extend `Model<Attrs, CreationAttrs, Ability?>`.
   - Base model owns `can()`, `iCan()`, `Opt()`, enum helpers, timestamp helpers.
   - Authorization-friendly return contract is `Able`.

2. **Structural Model Contract**
   - Each model declares `type Attrs`.
   - Each model defines `static attributes()`, `static initOptions()`, and `static async boot()`.
   - Factory export pattern is required: `export const X = () => ormModel(m => m.X) as typeof XModel;`.

3. **Owner and Polymorphism Contract**
   - Polymorphic owner pattern uses `owner_type` + `owner_id`.
   - Owner relations use `constraints: false` with owner scopes where needed.
   - Owner utility methods (`ownerIs`, `getOwner`) are used for explicit owner resolution.

4. **Ability Contract**
   - `Ability` type and `can()` override are required for actor-sensitive models.
   - `iCan()` is the canonical wrapper for consistent error and visual-mode behavior.
   - Requesters and bridge extras may consume `can()`, but model/requester remains source of truth.

5. **ORM to GQL Contract**
   - Any attr used by GraphQL filtering, sorting, extras, or relation loading must be represented in ORM attrs/options or required ORM attrs in bridges.
   - Bridge list queries must keep deterministic ordering and bounded list behavior.

## Accepted Patterns

- `DataTypes.JSONB` with typed metadata (`isTypedObject`) for structured payloads.
- Virtual attrs with getters and `requiredFields` where needed.
- Scoped associations for owner subtype access (`owner_type_*` scopes).
- Model helper functions that encapsulate domain logic and return strongly typed values.

## Optional Patterns

- Additional static helper methods if domain requires aggregate/time operations.
- Feature-specific scopes in `initOptions().scopes`.
- Rich bridge extras for capability hints (as long as mutation security stays in requester/model).

## Rejected Patterns

- Hardcoded principal IDs in production business flow.
- Unbounded root-many retrieval in GraphQL-facing paths.
- Direct mutation authorization in resolver only (without requester/model guard).
- Introducing non-Ejtmaa naming/typing style from unrelated projects.

## Canonical baseline

Ejtmaa style is the canonical modern baseline for this repository.

Use external projects only for idea extraction when they can be translated into the same contracts:

- same base model behavior (`Model.ts` contract),
- same bridge-driven GQL integration style,
- same actor/owner authorization model.

Any pattern that cannot map cleanly to these contracts is excluded.

## Compliance Entry Criteria for New Model Work

Before generating a model, confirm:

1. Actor or non-actor path is selected.
2. Ability contract requirements are identified.
3. Required GQL touchpoints are known.
4. Owner polymorphism need is explicitly decided.
5. Attrs/options/relations/helpers checklist is attached to the task.
