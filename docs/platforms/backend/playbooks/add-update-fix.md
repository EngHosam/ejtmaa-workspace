# Add / Update / Fix Playbook (Requester + GQL First)

## 1) Golden Workflow

For most backend feature work in Ejtmaa:
1. Design model contract (if data changes are required).
2. Add or update requester methods (write path).
3. Add or update GraphQL schema + bridge (read path).
4. Wire HTTP forms/data adapters if needed.
5. Regenerate GraphQL types.
6. Run type-check.

This is the default workflow unless explicitly overridden.

## 2) Add New Model + Requester + Bridge

### Step A: Model

1. Create model in `backend/src/app/orm/models`.
2. Define `Attrs`, `attributes()`, `initOptions()`, `boot()`.
3. Export factory function via `ormModel(...)`.
4. Add associations and typed mixins.
5. For JSONB fields, follow field-type rules (`isTypedObject` vs plain JSONB).

### Step B: Requester (Write Contract)

1. Create requester file under `backend/src/app/orchestrator/requesters`.
2. Add class decorator: `@requester("ident")`.
3. Add method decorators: `@sub([...platform], [...actors])`.
4. In each method:
   - validate with `withJoiFirewall`
   - use actor facts validators from `joi_rules.ts`
   - use `startTransaction()` for writes
   - return `SJsonResult`

Enum input rule in requesters:
- For enum-like inputs, use `joi.select(...)` (supports raw value and select-object payloads).
- Avoid raw free-text enum input contracts when project enum metadata already exists in models/translations.
- With `withJoiFirewall`, avoid redundant `.required()` for normal required fields; presence is already enforced by default unless explicitly overridden.

### Step C: HTTP Form Dispatcher

Use existing dispatch routes when possible:
- `/website/forms/requester/:requester/:sub`
- `/cpanel/forms/requester/:requester/:sub`

Usually no new dispatcher controller is needed; only requester registration is required.

### Step D: GraphQL Definition

1. Update role `.graphql` definitions (customer/supervisor as needed).
2. Keep `base.graphql` for shared scalar/interface/shared types.
3. Keep role query contracts explicit per role file.
4. Add relation fields only when expected relation count is `<= 100`.
5. If expected relation count is unknown, stop and clarify before adding relation fields.
6. Add relations only in the role schema that owns/needs them; do not mirror to other roles unless explicitly requested.
7. Do not add relations inside `type _Me` unless explicitly requested.
8. Use relation object fields (for example `customer`, `notification`) and avoid exposing `*_id` linkage columns.
9. Keep object field layout blocks ordered as:
   - `# info`
   - `# timestamps` (if present)
   - `# relations` (if present)
   - `# pagination` (if present)
10. For timestamped/paginated listable objects, use `implements _Timestamps & _Pagination`.
11. For enum filters, use value wrappers (`*_Value`) in input contracts.

### Step E: Bridge

1. Add bridge file in role-specific folder.
2. Define parent types (`GetManyParent`, `GetOneParent`) at top.
3. Set `static ident`, `static typeIdent`, `static ormModel`.
4. Override `getOrmFindOptions` for filtering/list behavior when needed.
5. Register bridge in role schema `registeredBridges`.
6. Do not re-declare ORM-native relations in `static relations`; rely on `BridgeBase` auto relation boot from model associations.
7. Use `lazy`/manual relation mapping only when explicitly requested and only for non-ORM derived relation paths.
8. For `me`-scoped root-one queries, prefer role base owner-parent resolution (`getRootOrmParent` in `*BridgeBase`) and avoid forcing `"STATIC"` in entity bridge overrides unless there is an explicit approved exception.
9. Root parent strategy is configurable (`owner parent`, `"STATIC"`, `{parent, as}`, scoped static); pick the least-complex strategy that satisfies scope and relation correctness.
10. Do not duplicate principal guard in `willPrepare` when the same guard is already enforced by role base `getRootOrmParent` for the active parent mode (for example `{me: true}`).
11. Keep `willPrepare` only when it introduces non-duplicated guard logic; `MeBridge` may keep it when base parent-mode guard is not the active enforcement path.
12. **SECURITY-CRITICAL:** duplicated guard logic across `willPrepare` and base `getRootOrmParent` is a review blocker unless there is an explicit, documented exception.

### Step F: Resolver

In schema class:
- Instantiate bridge with `AsRoot(...)`.
- Pass parent payload matching bridge parent types exactly.
- Return `prepareManyGQLModels(...)` or `prepareOneGQLModel(...)`.
- Keep typed signatures (`resolvers(): Resolvers<Context>`) instead of `any`.
- In complex cases, direct `prepareManyGQLModels(...)` / `prepareOneGQLModel(...)` orchestration is allowed, but must preserve `BridgeBase` internal relation/parent behavior.

### Step G: Verify

Run:
- `yarn generate-types`
- `yarn type-check`

If GraphQL contract/types are consumed by website:
- Sync files by command copy from backend gql source files to `website/src/types/gql/**` mirror paths.
- Do not hand-edit copied generated gql-types or copied definitions.

## 3) Update Existing Feature

When modifying behavior:
1. Update requester validation + execution first for write-path changes.
2. Update bridge query logic for read-path changes.
3. Update role GraphQL contracts.
4. Regenerate gql-types.
5. Ensure route actor guards still match feature intent.

Checklist:
- No decorator mismatch (`@requester` / `@sub`).
- No schema/type mismatch.
- No transaction leaks in multi-write logic.
- No controller business logic growth.

## 4) Fix / Debug Workflow

When a bug appears:
1. Identify failing layer first:
   - Route/middleware
   - Requester validation
   - Requester execution
   - Bridge filtering/include
   - GraphQL contract mismatch
2. Reproduce with smallest possible payload/query.
3. Patch at the true ownership layer (do not patch around symptoms).
4. Re-run codegen and type-check if GraphQL touched.

## 5) Constraints to Preserve

- Keep architecture consistent with existing repository patterns.
- Do not add cross-layer shortcuts.
- Keep decorators mandatory for requesters.
- Keep GraphQL role separation.
- Do not edit generated gql-types manually.
- Keep docs in English under `/docs`.

## 6) Owner Authorization Playbook (Customer / Supervisor)

Current codebase still relies mostly on facts + middleware for owner checks.
When feature complexity grows, apply this stronger pattern:
1. Define model `Ability` type.
2. Implement `can()` in owner model.
3. In requester mutation:
   - validate owner facts
   - load target model
   - call `owner.can(...)`
   - continue only on `"CAN"`

This gives reusable, auditable authorization behavior.

## 7) GraphQL Filtering Playbook

To add a list filter safely:
1. Add input/arg in role GraphQL definition.
2. Regenerate types.
3. Extend bridge parent type with optional filter field.
4. In `getOrmFindOptions`, merge filter into `where` (or include join filter).
5. Keep deterministic ordering.
6. Verify list + one query behavior independently.

For capability flags in GraphQL response:
1. Add bridge extra key (for example `canDelete`).
2. Implement in `loadExtra(...)` using owner `can(...)`.
3. Pass `visualMode: {lang: this.context.lang()}` when localized descriptions are needed.
4. Re-check authorization in requester writes even if extra returns `"CAN"`.

## 7.1) Supervisor customer read playbook (Ejtmaa)

Rule:
1. Add root query pairs (`customers` + `customer`) and `customerStats` on `SupervisorSchema`.
2. Keep resolvers thin with bridge parent payload only (`{public: true, filter}` or `{public: true, id}`).
3. Keep all search/scope filters in `getOrmFindOptions` on `CustomerBridge`.
4. Stats totals belong on `CustomerStatsBridge` (`total_count`), not `_Me`.
5. Use enum wrappers in filters (`*_Value`) and typed schema resolvers (`Resolvers<Context>`).

Ejtmaa reference:

- `backend/src/app/gql/schemas/SupervisorSchema.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts`

## 8) Scheduled Task Playbook

To add a production task:
1. Create task class.
2. Register in cron config.
3. Ensure idempotency.
4. Add logging around critical branches.
5. Add rollback-safe behavior for partial failures.
6. Verify in non-prod first.

## 9) Immediate Hardening Backlog

High-priority stabilization items:
1. Add focused tests for requester registration and actor-sub access matrix.
2. Add focused tests for owner `can()` checks on customer mutation paths.
3. Review and remove placeholder/example controllers when they are no longer needed operationally.

## 10) Definition of Done (Backend Task)

A backend task is complete only when:
- Business logic is in requester/bridge ownership layers.
- Actor and validation rules are explicit.
- GraphQL definitions and generated types are synchronized.
- `yarn generate-types` succeeds (when GraphQL changed).
- `yarn type-check` succeeds.
