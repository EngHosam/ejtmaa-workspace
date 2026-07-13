# GQL Worked Reference Pack

Concrete implementation references for the generator.

## 1) Schema example (role root)

### Target

- `backend/src/app/gql/schemas/CustomerSchema.ts`
- `backend/src/app/gql/schemas/SupervisorSchema.ts`

### Key moves

1. Define `Context` extending `RExe` with optional principal.
2. Register bridges in `registeredBridges`.
3. Keep `me` resolver bridge-driven (`MeBridge.AsRoot`).
4. Keep `notifications` query deterministic and bounded.

## 2) Base bridge example (role base contract)

### Target

- `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`
- `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts`

### Key moves

1. `schemaIdent`, `requiredOrmAttrs`, `registeredAttrs`, `maxLevel`.
2. `withListable` for pointer/page mode.
3. `withReplacements` for principal-aware binding.
4. `getRootOrmParent` for scoped `me` and `as` traversal.
5. `getOrmFindOptions` for deterministic many/one root paths.
6. `ignoreExtra` for list-safe performance behavior.

## 3) Entity bridge example (root one)

### Target

- `backend/src/app/gql/bridges/customer/MeBridge.ts`
- `backend/src/app/gql/bridges/supervisor/MeBridge.ts`

### Key moves

1. `ident`, `typeIdent`, `ormModel` defined.
2. `willPrepare` validates principal.
3. `getOrmFindOptions` resolves by principal id.

## 4) Supervisor customer list example

### Target

- `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts`

### Key moves

1. Map `_CustomerFilter` to Sequelize `where` and sort.
2. Serve `total_count` through `CustomerStatsBridge.loadExtra`.
3. Keep `SupervisorSchema` resolvers thin.

## 5) Union routing reference

### Target

- `backend/eng-hosam/@nodejs/gql/src/BridgeBase.ts`
- `docs/platforms/backend/patterns/gql-union-routing-reference.md`

### Key moves

1. Use `AsFragmentSub` and union include routing from framework base.
2. Ensure discriminators load through required attrs strategy.
3. Keep routing deterministic for each union member.

## 6) Minimum generator output for new feature

When generating a new GQL feature, include:

1. schema updates (`Query`, args, resolver delegation),
2. bridge updates (root parent + find options + mapping),
3. required attrs mapping notes,
4. checklist and quality-gate results.
