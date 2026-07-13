# GQL Pattern Atlas (Ejtmaa)

Reusable pattern families for Ejtmaa `schemas` and `bridges`.

## 1) Role-scoped schema partition

- **Problem**: avoid cross-role contract leakage.
- **Shape**: one schema class per role (`CustomerSchema`, `SupervisorSchema`).
- **Refs**: `backend/src/app/gql/schemas/CustomerSchema.ts`, `backend/src/app/gql/schemas/SupervisorSchema.ts`

## 2) Registered bridges as contract surface

- **Problem**: prevent implicit bridge access.
- **Shape**: `registeredBridges: [...]` in schema.
- **Refs**: `backend/eng-hosam/@nodejs/gql/src/GQLSchemaBase.ts`

## 3) Root bridge entry pattern

- **Problem**: keep resolver code thin.
- **Shape**: `Bridge.AsRoot(...).prepareManyGQLModels(...)` / `prepareOneGQLModel(...)`.
- **Refs**: `backend/src/app/gql/schemas/CustomerSchema.ts`, `backend/src/app/gql/schemas/SupervisorSchema.ts`

## 4) List window pattern

- **Problem**: unbounded list retrieval.
- **Shape**: `withListable()` with max clamp and offset policy.
- **Refs**: `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`, `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts`

## 5) Replacement binding pattern

- **Problem**: role-scoped query context propagation.
- **Shape**: `withReplacements()` using actor id placeholders (`customerId`, `supervisorId`).
- **Refs**: `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`, `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts`

## 6) MeBridge owner profile pattern

- **Problem**: secure current-actor profile retrieval.
- **Shape**: `willPrepare` enforces actor; `getOrmFindOptions` filters by actor id.
- **Refs**: `backend/src/app/gql/bridges/customer/MeBridge.ts`, `backend/src/app/gql/bridges/supervisor/MeBridge.ts`

## 7) Supervisor customer list pattern

- **Problem**: paginated supervisor customer administration.
- **Shape**: `CustomerBridge` maps `_CustomerFilter` to Sequelize `where` + sort; root `customerStats` via `CustomerStatsBridge`.
- **Refs**: `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`, `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts`, `backend/src/app/gql/definitions/supervisor.graphql`

## 8) Enum output wrapper pattern

- **Problem**: localized enum display in GQL reads.
- **Shape**: `_XValue` enum + `_X { value, label }` wrapper in `backend/src/app/gql/definitions/base.graphql`.
- **Refs**: `backend/src/app/gql/definitions/base.graphql`, role bridge base `loadAttr` / `toEnum`

## 9) HTTP-to-GQL adapter pattern

- **Problem**: frontend read transport.
- **Shape**: data adapter controller decodes `gql` query param and executes role schema.
- **Refs**: `backend/src/app/http/controllers/website/data_adapters/GQLAdapterController.ts`, `backend/src/app/http/controllers/cpanel/data_adapters/GQLAdapterController.ts`

## Related

- `docs/platforms/backend/patterns/gql-source-corpus.md`
- `docs/platforms/backend/patterns/graphql-and-bridges.md`
