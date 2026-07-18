# Plan Domain Contract (Current)

## 1) Scope

Current Ejtmaa plan (product catalog / الباقة) surface:

- ORM persistence for platform-wide pricing SKUs (`plans` table),
- customer GraphQL **public catalog** roots `plans` / `plan(id)`,
- localization for plan status and billing period,
- website GraphQL mirrors for that catalog read.

Out of scope (not shipped):

- `Subscription` / purchase lifecycle,
- `MyFatoorahInvoice` / payment session wiring,
- plan write requesters / mutations / supervisor CRUD GQL,
- seed rows for plans,
- SKU `code` column (explicitly rejected — identity is `id` + MultiLang `name`),
- naming the model `Package` (language-sensitive; use `Plan`),
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent).

## 2) Domain purpose

`Plan` is a **non-actor** platform catalog SKU: what the product sells (price, billing cadence, soft limits).

- Not owned by `Customer` or `Organization` (no tenant FK).
- Does **not** store payment or subscription state.
- Changing catalog price later must not rewrite historical purchases (those will snapshot on `Subscription` when that model ships).
- Customer GQL exposes **ACTIVE** plans only for list and detail.

Layering (future):

```text
Plan (catalog) ← Subscription (org purchase) ← MyFatoorahInvoice (gateway session)
```

## 3) ORM model

File: `backend/src/app/orm/models/Plan.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `plan`
- `tableName`: `plans`

Factory: `export const Plan = () => ormModel(m => m.Plan) …`

Runtime model registry: generated/local `.types/models.ts` must include `"Plan"` (gitignored; regenerate/update when adding models or `type-check` fails on factory).

### 3.1 Attrs layout

- `// identity` — `name`, `description`
- `// commercial` — `price`, `billing_period`, `billing_period_count`
- `// limits` — `max_members`, `max_meetings_per_month`
- `// lifecycle` — `sort_order`, `status`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `name` | JSONB | no | `MultiLangString` `{ ar, en }` |
| `description` | JSONB | yes | `MultiLangString` or null |
| `price` | DECIMAL(18, 2) | no | SAR (parsed via ORM DECIMAL → rounded float) |
| `billing_period` | STRING(191) | no | enum `planBillingPeriod` |
| `billing_period_count` | INTEGER | no | default `1` |
| `max_members` | INTEGER | yes | null = unlimited |
| `max_meetings_per_month` | INTEGER | yes | null = unlimited |
| `sort_order` | INTEGER | no | default `0` |
| `status` | STRING(191) | no | enum `planStatus`, default `ACTIVE` |

No `code` / SKU string column.

### 3.3 Billing period (not day counts)

Commercial duration uses calendar period + count:

- `billing_period`: `MONTHLY` \| `YEARLY`
- `billing_period_count`: usually `1` (allows future multi-period SKUs without schema reshape)

Do **not** store `duration_days` for plan length — renewal math belongs on subscription using calendar months/years.

### 3.4 Indexes

- `plans_status_sort_order` — (`status`, `sort_order`)

### 3.5 Relations

`boot()` is empty today. No `hasMany` Subscription association until that model ships.

### 3.6 Enums (translations)

Keys under `backend/src/resources/trans/{ar,en}/general.ts` → `enums`:

| Enum key | Values |
|---|---|
| `planStatus` | `ACTIVE`, `DISABLED` |
| `planBillingPeriod` | `MONTHLY`, `YEARLY` |

ORM attrs use `enum: ["one", "planStatus"]` / `["one", "planBillingPeriod"]` with `isTypedObject: true`.

## 4) GraphQL (customer)

### 4.1 SDL

- `backend/src/app/gql/definitions/base.graphql` — `_PlanStatus` / `_PlanStatusValue`, `_PlanBillingPeriod` / `_PlanBillingPeriodValue`
- `backend/src/app/gql/definitions/customer.graphql` — type `_Plan`, roots `plans`, `plan(id: ID!)`

### 4.2 Type `_Plan`

Implements `_Timestamps` & `_Pagination`.

| Field | GQL type | Mapping notes |
|---|---|---|
| `id` | `ID!` | |
| `name` | `String` | MultiLang JSONB → localized via `CustomerBridgeBase.loadAttr` |
| `description` | `String` | same |
| `price` | `Float` | |
| `billing_period` | `_PlanBillingPeriod` | enum wrapper `{ value, label }` |
| `billing_period_count` | `Int` | |
| `max_members` | `Int` | |
| `max_meetings_per_month` | `Int` | |
| `sort_order` | `Int` | |
| `status` | `_PlanStatus` | enum wrapper |
| `created_at` / `updated_at` | `String!` | |
| `total_count` | `Int` | pagination |

No nested relations on `_Plan` yet.

### 4.3 Bridge

File: `backend/src/app/gql/bridges/customer/PlanBridge.ts`

- Extends `CustomerBridgeBase` (not `CustomerOrganizationOwnedBridgeBase`)
- `ident = "plan"`, `typeIdent = "_Plan"`, `ormModel = PlanModel`
- Parent payloads:
  - many: `{ public: true }`
  - one: `{ public: true; id: string }`
- `getRootOrmParent`: inherited → `"STATIC"` (no owner resolve)
- `getOrmFindOptions` entity policy:
  - many: `where: { status: "ACTIVE" }`, `order: [["sort_order","asc"],["id","asc"]]`, plus `withListable` / `withReplacements`
  - one: `where: { id, status: "ACTIVE" }`

### 4.4 Schema resolvers

`backend/src/app/gql/schemas/CustomerSchema.ts`:

- register `PlanBridge`
- `plans` → `prepareManyGQLModels({ public: true })`
- `plan` → `prepareOneGQLModel({ public: true, id })`

Supervisor schema: **no** Plan surface.

### 4.5 Authz / visibility

- Catalog roots are not actor-scoped; any caller with customer schema access can read ACTIVE plans.
- `DISABLED` plans are omitted from both list and detail (detail by id of a disabled plan returns empty/not found per bridge `where`).
- Write authorization for catalog management is not shipped.

## 5) Website mirrors

| Platform | Status |
|---|---|
| `website/` | Active — copy `base` + `customer` SDL/types after backend `yarn generate-types` |
| `cpanel/` | Deferred — no Plan supervisor SDL yet; do not invent mirrors |

Mirror paths:

- `website/src/types/gql/definitions/base.graphql`
- `website/src/types/gql/definitions/customer.graphql`
- `website/src/types/gql/gql-types/base.ts`
- `website/src/types/gql/gql-types/customer.ts`

## 6) Naming and seed collisions

- ORM/GQL English name: **`Plan`** / `_Plan` / `plans` — Arabic product name remains الباقة.
- Do **not** rename to `Package` (reserved words / package managers collide).
- Seed helper arrays must **not** be named `plan` / `plans` / `SeedPlan` — see `.cursor/rules/backend-demo-seed-conventions.mdc`.

## 7) Verification

Existing scripts only:

- `backend/`: `yarn generate-types`, `yarn type-check`
- `website/`: `yarn type-check` after mirror copy

## 8) Traceability map

| Path | Status | Doc home |
|---|---|---|
| `backend/src/app/orm/models/Plan.ts` | added | §3 |
| `backend/src/app/gql/bridges/customer/PlanBridge.ts` | added | §4.3 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | modified | §4.4 |
| `backend/src/app/gql/definitions/base.graphql` | modified | §4.1 |
| `backend/src/app/gql/definitions/customer.graphql` | modified | §4.1–4.2 |
| `backend/src/app/gql/gql-types/base.ts` | generated | §4 / §7 |
| `backend/src/app/gql/gql-types/customer.ts` | generated | §4 / §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | generated (shared base enums; no Plan queries) | §4 — no supervisor Plan surface |
| `backend/src/resources/trans/ar/general.ts` | modified | §3.6 |
| `backend/src/resources/trans/en/general.ts` | modified | §3.6 |
| `backend/.types/models.ts` | local/gitignored | §3 — `"Plan"` registry entry |
| `website/src/types/gql/definitions/base.graphql` | mirror | §5 |
| `website/src/types/gql/definitions/customer.graphql` | mirror | §5 |
| `website/src/types/gql/gql-types/base.ts` | mirror (generated) | §5 |
| `website/src/types/gql/gql-types/customer.ts` | mirror (generated) | §5 |
| `website/lib/tsconfig.tsbuildinfo` | build artifact | excluded from narrative |
| `.cursor/skills/orm-model-generator/SKILL.md` | modified | Ejtmaa addendum `Plan` |
| `.cursor/skills/gql-schema-bridge-generator/SKILL.md` | modified | Ejtmaa addendum `plans`/`plan` |
| `.cursor/rules/gql-root-parent-payload-contract.mdc` | modified | `{ public: true }` catalog roots |
| `.cursor/rules/backend-demo-seed-conventions.mdc` | modified | seed name vs `Plan` model |
| `.cursor/rules/plan-catalog-domain.mdc` | added | durable Plan invariants |
| `docs/platforms/backend/contracts/member-domain.md` | modified | seed naming note → `Plan` |
| `docs/platforms/backend/patterns/scheduler-console-seed-db.md` | modified | seed naming note → `Plan` |
| `docs/platforms/backend/contracts/plan-domain.md` | this file | all |

## 9) Related

- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md` (payment mount target; not yet wired to Plan)
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `.cursor/rules/plan-catalog-domain.mdc`
