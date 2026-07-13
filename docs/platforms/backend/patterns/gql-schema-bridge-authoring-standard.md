# GQL Schema and Bridge Authoring Standard (Ejtmaa)

## Scope Lock

- **Primary style**: Ejtmaa.
- **Primary style**: Ejtmaa backend `gql/` tree only.
- **Exclusion**: any drifted contract from blacklist/refusal matrix.

## Domain Contracts

1. **Role boundary**: each role owns its schema and allowed bridge set.
2. **Schema ownership**: bridge `schemaIdent` must match schema `ident`.
3. **Principal contract**: context must define principal fields, `lang`, list controls, `by`, `originalQuery`.
4. **Parent modes**: each bridge must explicitly declare supported root parent modes (`me`, `public`, `self`, `STATIC`, aliases).
5. **Polymorphism**: union/object-owner routing must be deterministic and discriminator-driven.
   - Use `docs/platforms/backend/patterns/gql-union-routing-reference.md` as the canonical runtime reference.

## Function Contracts

1. **Resolver policy**
   - model-backed reads use bridge pipeline (`AsRoot` + `prepare...`).
   - non-bridge resolver path requires explicit rationale.
   - role schemas should use generated resolver typing (`resolvers(): Resolvers<Context>`), not `any`.
2. **Bridge policy**
   - define list/parent/query/depth/required-attrs/extras strategy.
   - root-many always bounded and ordered.
   - rely on ORM association auto-bridging by default; do not re-declare ORM-native relations in `static relations`.
   - use manual relation mapping (`lazy`/custom tuples) only for explicitly requested non-ORM derived relations.
3. **Mapping policy**
   - any sort/filter/extra/union attr must be selected by ORM (`requiredOrmAttrs`/include attrs).
   - filter enum fields should use explicit value wrappers in SDL (`*_Value`) per enum wrapper contract.
4. **Security policy**
   - principal checks happen before scoped fetch (`willPrepare` / `getRootOrmParent` / scoped `where`).
   - `STATIC` actor-bound root-one must include ownership scope in `where`.
   - for `me`-scoped root-one where role base already resolves owner parent, prefer owner-parent path and avoid forcing `"STATIC"` in entity bridge overrides.
   - avoid duplicate principal guards: if role base `getRootOrmParent` already enforces principal for the parent mode, do not repeat the same check in entity `willPrepare`.
5. **Query safety**
   - no user-input interpolation in `sequelize.literal`.
   - use operators and parameterized binds/replacements.

## Role Bridge Base Contract (Mandatory)

Every role bridge base (`CustomerBridgeBase`, `SupervisorBridgeBase`, future role bases) must implement:

1. **`loadAttr` override** — resolves MultiLang JSONB to localized strings, MultiLang arrays, and enum attrs to `{value, label, icon}`.
2. **`trans` helper** — delegates to `@nodejs/localization` with `context.lang()`.
3. **`toEnum` / `toManyEnums`** — typed enum resolution using translation enums.
4. **`willPrepare` base** — empty default for entity bridge overrides.

Without `loadAttr`, JSONB fields declared as `String` in SDL cause serialization errors that abort entire queries via `GQLSchemaBase.execute`.

Full specification: `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`.

## Expansion Contracts

1. **Schema growth**
   - update `.graphql` + generated types + resolver parent shapes together.
2. **Bridge growth**
   - register bridge in schema, define `ident/typeIdent/ormModel`, verify relation/union mapping.
3. **Extras growth**
   - heavy ability/extras logic one-view by default.
4. **Depth growth**
   - keep finite `maxLevel` and explicit `deepInclude`.
5. **Framework integrity**
   - validate boot/registration invariants from `BridgeBase` and `GQLSchemaBase`.
   - verify schema construction contract remains compatible with provider invocation.

## Decision Trees

### A) Add Query Field

1. model-backed?
   - yes -> bridge path
   - no -> explicit service resolver rationale
2. root-many?
   - yes -> clamp + order mandatory
3. scoped?
   - yes -> principal guard + ownership filtering

### B) Add Bridge

1. define `ident/typeIdent/ormModel`
2. register in owning schema
3. choose parent mode strategy
4. define many/one options
5. verify mapping attrs
6. verify security and depth gates

### C) Add Filter

1. update GraphQL args/types
2. map args to `where/include/order`
3. verify selected attrs
4. enforce query safety (no unsafe literals)
5. run pagination+auth regression checks

### D) Add Union

1. define union members + required discriminators
2. implement exhaustive routing
3. validate role-specific schema visibility
4. validate against `docs/platforms/backend/patterns/gql-union-routing-reference.md`

## Hard Policies (Non-Negotiable)

- no unbounded root-many.
- no missing principal guard on scoped paths.
- no unresolved ORM attr dependency.
- no unsafe SQL literal interpolation.
- no non-Ejtmaa drift by copy-paste.

## Merge Checklist

1. `gql-source-corpus.md` compatibility check.
2. `gql-pattern-atlas.md` pattern selection.
3. cookbook playbook selection (if complex).
4. blacklist scan clean.
5. refusal matrix not triggered.
6. evaluation suite pass with accepted rubric score.

## Definition of Done

Accept only when:

- domain/function/expansion contracts all pass,
- security + mapping + pagination + depth gates pass,
- output is reproducible through skill + AGENTS runbook.
