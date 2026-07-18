# Subscription Domain Contract (Current)

## 1) Scope

Current Ejtmaa subscription (الاشتراك) surface:

- ORM persistence for customer-owned entitlement rows (`subscriptions` table),
- snapshot of catalog commercial terms at purchase time,
- static `Subscription.subscribe` for paid entitlement create (replace prior `ACTIVE`),
- customer GraphQL reads: roots `subscriptions` / `subscription(id)`, nested `_Me.currentSubscription` and `_Subscription.plan`,
- `_Me.canSubscribe(planId)` ability exposure (policy on `Customer.can`),
- hourly scheduler promotion of overdue `ACTIVE` rows to `EXPIRED`,
- website GraphQL mirrors for those reads,
- localization for `subscriptionStatus`.

Payment session create + finalize live in `myfatoorah-invoice-payment-domain.md` (requester `subscription.subscribe` starts checkout; finalize calls `Subscription.subscribe`).

Out of scope (not shipped):

- renew-specific requester beyond `subscribe`,
- supervisor Subscription GQL / cpanel mirrors,
- seed rows for subscriptions,
- statuses `PENDING` / `CANCELED` on Subscription (checkout belongs on `MyFatoorahInvoice`; no subscription row before entitlement).

## 2) Domain purpose

`Subscription` is a **non-actor** entitlement row owned by **Customer** (payer), not Organization.

- Real FK `customer_id` → Customer.
- Catalog link `plan_id` → Plan.
- Snapshots commercial terms so later Plan price/limit edits do not rewrite history.
- Billing period is chosen at subscribe time and stored on the subscription (`plan_billing_period` + `plan_billing_period_count`); Plan holds both list prices.
- Natural end → `EXPIRED` (scheduler). Mid-cycle replace/renew: insert a **new** row via `Subscription.subscribe`; previous `ACTIVE` → `REPLACED`.

Layering:

```text
Plan (catalog) ← Subscription (customer entitlement) ← MyFatoorahInvoice (gateway session)
```

## 3) ORM model

File: `backend/src/app/orm/models/Subscription.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `subscription`
- `tableName`: `subscriptions`

Factory: `export const Subscription = () => ormModel(m => m.Subscription) …`

Runtime model registry: generated/local `.types/models.ts` must include `"Subscription"` (gitignored; regenerate/update when adding models or `type-check` fails on factory).

### 3.1 Attrs layout

- `//relations` — `customer_id`, `plan_id`
- `//info` — snapshot + window + `status`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `customer_id` | BIGINT | no | payer Customer |
| `plan_id` | BIGINT | no | catalog Plan |
| `plan_price` | DECIMAL(18, 2) | no | snapshotted SAR amount for chosen period |
| `plan_billing_period` | STRING(191) | no | enum `planBillingPeriod` (`MONTHLY` \| `YEARLY`) |
| `plan_billing_period_count` | INTEGER | no | default `1` |
| `plan_max_members` | INTEGER | yes | snapshot; null = unlimited |
| `plan_max_meetings_per_month` | INTEGER | yes | snapshot; null = unlimited |
| `starts_at` | DATE | no | entitlement window start |
| `ends_at` | DATE | no | entitlement window end |
| `status` | STRING(191) | no | enum `subscriptionStatus`, default `ACTIVE` |

No payment gateway columns on this model.

### 3.3 Statuses

| Value | Meaning |
|---|---|
| `ACTIVE` | Current entitlement (may still be past `ends_at` until scheduler runs) |
| `EXPIRED` | Natural end (`ends_at <= now`, set by task or future writer) |
| `REPLACED` | Superseded by a newer subscription row (`Subscription.subscribe`) |

Do **not** reintroduce `PENDING` / `CANCELED` without an explicit product decision.

### 3.4 Indexes

- `subscriptions_customer_id` — (`customer_id`)
- `subscriptions_plan_id` — (`plan_id`)
- `subscriptions_customer_id_status` — (`customer_id`, `status`)

### 3.5 Scopes

`scopes.current` is a **function** (fresh `new Date()` each use):

- `status = ACTIVE`
- `ends_at > now`

Used by `Customer.hasOne(Subscription().scope("current"), { as: "currentSubscription" })`.

### 3.6 Relations

| Side | Association |
|---|---|
| Subscription | `belongsTo(Customer)`, `belongsTo(Plan)` |
| Customer | `hasMany(Subscription)`; `hasOne(Subscription.scope("current"), { as: "currentSubscription" })` |
| Plan | `hasMany(Subscription)` |

### 3.7 Enums (translations)

Keys under `backend/src/resources/trans/{ar,en}/general.ts` → `enums`:

| Enum key | Values | Used on |
|---|---|---|
| `subscriptionStatus` | `ACTIVE`, `EXPIRED`, `REPLACED` | Subscription ORM / GQL |
| `planBillingPeriod` | `MONTHLY`, `YEARLY` | Subscription snapshot (shared with Plan domain docs) |

### 3.8 Write path policy

- Entitlement create: static `Subscription.subscribe(op)` — snapshots Plan limits, sets calendar `starts_at`/`ends_at` from billing period, marks prior customer `ACTIVE` rows `REPLACED`, inserts new `ACTIVE`.
- Callers (payment finalize, future requesters) must use that static; do not duplicate replace+create.
- Scheduler may bulk-update status only (see §5).

## 4) GraphQL (customer)

### 4.1 SDL

- `backend/src/app/gql/definitions/base.graphql` — `_SubscriptionStatus` / `_SubscriptionStatusValue`
- `backend/src/app/gql/definitions/customer.graphql` — type `_Subscription`, roots `subscriptions` / `subscription(id)`, `_Me.currentSubscription`, `_Me.canSubscribe(planId)`, `_Subscription.plan`

### 4.2 Type `_Subscription`

Implements `_Timestamps` & `_Pagination`.

| Field | GQL type | Mapping notes |
|---|---|---|
| `id` | `ID!` | |
| `plan_price` | `Float` | |
| `plan_billing_period` | `_PlanBillingPeriod` | enum wrapper |
| `plan_billing_period_count` | `Int` | |
| `plan_max_members` | `Int` | |
| `plan_max_meetings_per_month` | `Int` | |
| `starts_at` / `ends_at` | `String` | |
| `status` | `_SubscriptionStatus` | enum wrapper |
| `created_at` / `updated_at` | `String!` | |
| `plan` | `_Plan` | nested via PlanBridge |
| `total_count` | `Int` | pagination |

No FK scalars (`customer_id` / `plan_id`) on the GQL type.

### 4.3 Bridge

File: `backend/src/app/gql/bridges/customer/SubscriptionBridge.ts`

- Extends `CustomerBridgeBase` (not org-owned base)
- `ident = "subscription"` (must match ORM `modelName`)
- `typeIdent = "_Subscription"`, `ormModel = SubscriptionModel`
- Parent payloads:
  - many root: `{ me: true }` → root parent = context Customer (association `subscriptions`)
  - one root: `{ me: true; id: string }`
  - nested from Me: parent `CustomerModel` (`currentSubscription`)
  - model instance when already loaded
- List order: `starts_at` desc, then `id` desc
- Nested `_Subscription.plan`: `PlanBridge.GetOneParent` includes `SubscriptionModel`

### 4.4 `_Me.currentSubscription` wiring

Do **not** add a second bridge class (e.g. `CurrentSubscriptionBridge`).

Mechanism:

1. ORM: `Customer.hasOne(..., { as: "currentSubscription" })` registers association key `currentSubscription`; target `modelName` = `subscription`.
2. `MeBridge` `bootRelations` auto-builds `["currentSubscription", ["one", "subscription"]]`.
3. Schema `mappedRegisteredBridges["subscription"]` resolves to `SubscriptionBridge`.

When to declare `MeBridge.static relations` instead: only if there is **no** Sequelize association (custom getter) or you need non-default options (e.g. `"lazy"`). Sibling reference: qline company MeBridge `activeCompanySubscription` (explicit + lazy because no `hasOne`).

### 4.5 Schema resolvers

`backend/src/app/gql/schemas/CustomerSchema.ts`:

- register `SubscriptionBridge` only (not a current-subscription bridge)
- `subscriptions` → `prepareManyGQLModels({ me: true })`
- `subscription` → `prepareOneGQLModel({ me: true, id })`

Supervisor schema: **no** Subscription surface.

### 4.6 Authz / visibility

- Roots require customer principal (`{ me: true }` → `NOT_PERMIT` without `context.customer`).
- Scope is the current customer's association / ownership via Customer as root parent.
- `currentSubscription` returns at most one ACTIVE + not-ended row (scope), or null.
- Catalog Plan nested under subscription is readable as relation of an owned subscription row.

## 5) Scheduler

File: `backend/src/app/scheduler/tasks/ExpireSubscriptionsTask.ts`  
Registered in: `backend/src/resources/configs/scheduler/cron.ts`

| Property | Value |
|---|---|
| Schedule | every 1 hour |
| `autoStart` | `true` |
| `runOnInit` | `false` |
| Effect | bulk `update` `ACTIVE` + `ends_at <= now` → `EXPIRED` |

Idempotent: re-running does not change already `EXPIRED` rows.

Until the task runs, an `ACTIVE` row with past `ends_at` may still appear in history roots; it will **not** match `current` scope (`ends_at > now`).

## 6) Seeds

No subscription seed rows.

Plan catalog seed (`seedDemoCatalogPlans` / `demoCatalogPlanEntries`) remains under Plan domain / `scheduler-console-seed-db.md` — not subscription.

## 7) Website mirrors

| Platform | Status |
|---|---|
| `website/` | Active — copy `base` + `customer` SDL/types after backend `yarn generate-types` |
| `cpanel/` | Deferred — no supervisor Subscription surface |

Mirror paths:

- `website/src/types/gql/definitions/base.graphql`
- `website/src/types/gql/definitions/customer.graphql`
- `website/src/types/gql/gql-types/base.ts`
- `website/src/types/gql/gql-types/customer.ts`

## 8) Verification

Existing scripts only:

- `backend/`: `yarn generate-types`, `yarn type-check`
- `website/`: `yarn type-check` after mirror copy

## 9) Traceability map

| Path | Status | Doc home |
|---|---|---|
| `backend/src/app/orm/models/Subscription.ts` | added | §3 |
| `backend/src/app/orm/models/Customer.ts` | modified | §3.6 / §4.4 |
| `backend/src/app/orm/models/Plan.ts` | modified | §3.6; plan-domain |
| `backend/src/app/gql/bridges/customer/SubscriptionBridge.ts` | added | §4.3 |
| `backend/src/app/gql/bridges/customer/PlanBridge.ts` | modified | §4.3 nested plan parent |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | modified | §4.5 |
| `backend/src/app/gql/definitions/base.graphql` | modified | §4.1 |
| `backend/src/app/gql/definitions/customer.graphql` | modified | §4.1–4.2 |
| `backend/src/app/gql/gql-types/base.ts` | generated | §4 / §8 |
| `backend/src/app/gql/gql-types/customer.ts` | generated | §4 / §8 |
| `backend/src/app/gql/gql-types/supervisor.ts` | generated (shared base enums; no Subscription queries) | §4.5 |
| `backend/src/app/scheduler/tasks/ExpireSubscriptionsTask.ts` | added | §5 |
| `backend/src/resources/configs/scheduler/cron.ts` | modified | §5 |
| `backend/src/resources/trans/ar/general.ts` | modified | §3.7 |
| `backend/src/resources/trans/en/general.ts` | modified | §3.7 |
| `backend/src/app/orm/patches/SeedPatch.ts` | modified (Plan seed only) | §6 — no subscription seed |
| `backend/.types/models.ts` | local/gitignored | §3 — `"Subscription"` registry entry |
| `website/src/types/gql/definitions/base.graphql` | mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | mirror (generated) | §7 |
| `website/src/types/gql/gql-types/customer.ts` | mirror (generated) | §7 |
| `website/lib/tsconfig.tsbuildinfo` | build artifact | excluded from narrative |
| `.cursor/skills/orm-model-generator/SKILL.md` | modified | Ejtmaa addendum `Subscription` |
| `.cursor/skills/gql-schema-bridge-generator/SKILL.md` | modified | Ejtmaa addendum subscriptions / Me auto-wire |
| `.cursor/rules/subscription-domain.mdc` | added | durable Subscription invariants |
| `.cursor/rules/gql-association-auto-relations.mdc` | added | MeBridge association → bridge `ident` |
| `docs/platforms/backend/contracts/subscription-domain.md` | this file | all |

## 10) Related

- `docs/platforms/backend/contracts/plan-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/patterns/scheduler-console-seed-db.md`
- `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md`
- `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `.cursor/rules/subscription-domain.mdc`
- `.cursor/rules/myfatoorah-invoice-payment-domain.mdc`
- `.cursor/rules/gql-association-auto-relations.mdc`
