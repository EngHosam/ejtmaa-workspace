# GraphQL and Bridge Patterns

## 1) Core Principle

GraphQL in Ejtmaa is role-scoped and bridge-driven:
- Each actor role has its own schema class and definition file.
- Bridges are the ORM-to-GQL mapping layer.
- Requesters remain the mutation/write backbone; GraphQL currently focuses on query retrieval.

## 2) Schema Topology

Provider config:
- `backend/src/resources/configs/gql/index.ts`

Role schema classes:
- `backend/src/app/gql/schemas/CustomerSchema.ts`
- `backend/src/app/gql/schemas/SupervisorSchema.ts`

Definition files:
- `backend/src/app/gql/definitions/base.graphql`
- `backend/src/app/gql/definitions/shared.graphql`
- `backend/src/app/gql/definitions/customer.graphql`
- `backend/src/app/gql/definitions/supervisor.graphql`

Customer role query contract:
- `me`
- `notifications`

Supervisor role query contract:
- `me`
- `notifications`
- `customers` / `customer`
- `customerStats`

## 3) Role Isolation Rules

Follow role isolation discipline:
- Keep role schemas independent.
- Add type/query updates in all affected role files explicitly.
- Do not rely on cross-role implicit schema inheritance.

In current Ejtmaa:
- Each role file declares `type Query` with its `notifications` field.
- `backend/src/app/gql/definitions/shared.graphql` on disk mirrors the notification query sketch; role SDL files are authoritative in codegen.

## 4) Bridge Base Classes

Role-specific bridge bases:
- `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`
- `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts`

Common behavior included in both:
- `withListable()`:
  - Pointer pagination (`pointer`, `maxLength`).
  - Page pagination (`pagination`, `page`).
  - Safe max length clamp (`<= 50`).
- `withReplacements()`:
  - Role-specific replacement IDs (`customerId`, `supervisorId`).
- `withUpOnly()`:
  - Incremental ID-based slicing via `upOnly.lastId`.
- `deepInclude()`:
  - Include depth cutoff (`level <= 2`).
- `getRequiredOrmAttrs()`:
  - Return `*` for root single loads.
- `getRootOrmParent()`:
  - Supports `{me: true}` ownership parenting.
  - Supports `{me: true, as: "relationName"}` association parenting.
- `getOrmFindOptions()`:
  - Root many: listable + replacements + `order: [["updated_at", "desc"]]`.
  - Root one: `where.id` when parent includes `id`.
- Thin entity bridges may rely entirely on these defaults while still requiring an explicit `{ public: true, ... }` parent payload from resolvers for contract clarity; see `.cursor/rules/gql-root-parent-payload-contract.mdc`.
- `ignoreExtra()`:
  - Suppresses extras unless `prepareType === "one"`.

## 5) MeBridge Pattern

Current registered bridge in each role is `MeBridge`:
- `backend/src/app/gql/bridges/customer/MeBridge.ts`, `backend/src/app/gql/bridges/supervisor/MeBridge.ts`

Behavior:
1. `willPrepare` enforces role actor presence in context.
2. `getOrmFindOptions` filters by current actor ID.
3. Bridge executes as root and returns one model.

This is the minimal secure pattern for owner profile retrieval.

## 6) Filtering Pattern (Important)

Even though current role contracts are small, the filtering foundation is already in place.

### Existing Generic Filters
- Cursor/list slicing through pointer/page inputs.
- Root one filtering by `id`.
- Optional `upOnly` filtering for incremental clients.

### Recommended filter pattern

When adding query filters:
1. Extend role `.graphql` input/query args.
2. Regenerate gql types.
3. Extend bridge parent types (`GetManyParent`, `GetOneParent`).
4. Override `getOrmFindOptions` and map filter args into Sequelize `where/include`.
5. Keep resolver parent payload and bridge parent types fully aligned.

Do not place filtering logic directly in resolver bodies if the bridge is the model owner.

## 6.1) Enum Output Wrapper Pattern (Mandatory)

Use enum wrappers in GraphQL read contracts:
- In `backend/src/app/gql/definitions/base.graphql`:
  - define `enum _XValue { ... }`
  - define `type _X { value: _XValue!, label: String }`
- In role output types:
  - use `_X` wrapper for enum-like output fields.
- In role filter inputs:
  - use `_XValue`.

Do not expose raw enum output fields directly where wrapper object contract is expected.

## 7) Registration invariant

Schema classes use `registeredBridges`.

Rule: any bridge needed by relation fields must be registered, even when no direct root query calls it.

Current Ejtmaa registration:

| Schema | Bridges |
|---|---|
| `CustomerSchema` | `MeBridge`, `NotificationBridge` |
| `SupervisorSchema` | `MeBridge`, `NotificationBridge`, `CustomerBridge`, `CustomerStatsBridge` |

## 8) Resolver pattern

In all role schemas:

- `me` resolves via `MeBridge.AsRoot(...).prepareOneGQLModel(...)`.
- `notifications` resolves via `NotificationBridge` list contract.
- `customers` / `customer` / `customerStats` resolve via supervisor bridges only.

Keep complex model query trees in bridges. Use direct resolvers only for non-bridge service operations or simple bounded fetches.

## 9) Ability checks inside bridges (`can()` pattern)

Bridge-level `can()` exposes read-time capability hints to the UI. Mutation security remains in requester `throwMode` enforcement.

Ejtmaa reference bridges:

- `backend/src/app/gql/bridges/customer/MeBridge.ts`
- `backend/src/app/gql/bridges/supervisor/MeBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts`

Current Ejtmaa status: owner `can()` is active at model/requester level. Bridge extras are minimal on `MeBridge` only.

## 10) HTTP-to-GraphQL Adapter Pattern

Data adapter controllers execute arbitrary GraphQL query strings via query param:
- Decode `gql` from URL.
- Pass runtime vars.
- Inject role owner into schema execution context.
- Normalize output to `{section, records, pointer metadata}`.

Files:
- Website: `backend/src/app/http/controllers/website/data_adapters/GQLAdapterController.ts`
- Cpanel: `backend/src/app/http/controllers/cpanel/data_adapters/GQLAdapterController.ts`

## 11) Codegen and Type Contracts

Codegen config:
- `backend/codegen.yml`

Generated files:
- `backend/src/app/gql/gql-types/base.ts`
- `backend/src/app/gql/gql-types/customer.ts`
- `backend/src/app/gql/gql-types/supervisor.ts`

Rules:
- Do not edit generated gql-types manually.
- Always run `yarn generate-types` after changing `.graphql` definitions.
- Follow with `yarn type-check`.

## 12) Constraints and Guardrails

- Keep GraphQL contracts synchronized with bridge/model fields.
- Keep role contexts strict (`visitor`, `customer`, `supervisor`).
- Do not bypass actor checks in `willPrepare`.
- Avoid unbounded includes/deep nested relations (preserve `deepInclude` constraints).
- Prefer deterministic sorting for list queries.

## 13) Current Practical Constraints

- `_Notification.type` and `_Notification.identify` are scalar `String` fields in `backend/src/app/gql/definitions/base.graphql`, matching resolver behavior.
- `backend/src/app/gql/definitions/shared.graphql` on disk mirrors the notification query sketch; role SDL files are authoritative in codegen.
- Schema classes include local fallback context loading (`findByPk(1)`), but production usage should rely on HTTP adapters that inject authenticated role context.

## 14) Related Governance Documents

For ORM model generation and ORM-to-GQL mapping governance, also follow:

- `docs/platforms/backend/patterns/orm-model-baseline-audit.md`
- `docs/platforms/backend/patterns/orm-model-authoring-standard.md`
- `docs/platforms/backend/patterns/orm-model-compliance-checklists.md`
- `docs/platforms/backend/patterns/orm-model-worked-examples.md`
- `.cursor/skills/orm-model-generator/SKILL.md`

## 15) GQL Constitution Package

For strict schema/bridge generation and review, follow:

- `docs/platforms/backend/patterns/gql-constitution-map.md`
- `docs/platforms/backend/patterns/gql-source-corpus.md`
- `docs/platforms/backend/patterns/gql-pattern-atlas.md`
- `docs/platforms/backend/patterns/gql-schema-bridge-authoring-standard.md`
- `docs/platforms/backend/patterns/gql-complex-problems-cookbook.md`
- `docs/platforms/backend/patterns/gql-complex-problems-playbooks.md`
- `docs/platforms/backend/patterns/gql-anti-patterns-blacklist.md`
- `docs/platforms/backend/patterns/gql-refusal-matrix.md`
- `docs/platforms/backend/patterns/gql-worked-reference-pack.md`
- `docs/platforms/backend/patterns/gql-evaluation-suite.md`
- `docs/platforms/backend/patterns/gql-scoring-rubric.md`
- `.cursor/skills/gql-schema-bridge-generator/SKILL.md`
