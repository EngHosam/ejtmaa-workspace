# Supervisor Catalog, Meetings, and Home GraphQL

## 1) Scope

Supervisor GraphQL surfaces added after the customers directory, consumed by `cpanel/`:

- catalog: `plans` / `plan` / `planStats`
- payer rows: `subscriptions` / `subscription` / `subscriptionStats`
- platform meetings: `meetings` / `meeting` / `meetingStats` plus slim nested `_Member` (chairperson)
- dashboard payload: `home` (`HomeBridge` extras + month series)

SDL: `backend/src/app/gql/definitions/supervisor.graphql`.  
Schema: `backend/src/app/gql/schemas/SupervisorSchema.ts` (thin `AsRoot` + `prepareOne` / `prepareMany`).

Customer list/stats remain in `supervisor-customers-and-stats.md`. Organization roots remain in `organization-domain.md`.

## 2) Product language

| Arabic UI | English domain |
|---|---|
| باقة | Plan |
| اشتراك | Subscription |
| مؤسسة | Organization (nested copy on customer/meeting cards) |

Do not invent revenue, conversion, or time-series that the ORM does not store.

## 3) Plans

Query: `plans(filter: _PlanFilter)`, `plan(id: ID!)`, `planStats`.

`_PlanFilter`: `search`, `status` (`_PlanStatusValue`). Search is iLike on MultiLang `name` (`ar` and `en` JSONB paths). Description is not searched. Cpanel list UI currently sends `search` only.

`_PlanStats` extras (`PlanStatsBridge`): `total_count`, `active_count` (`status: "ACTIVE"`).

Writes are **not** GraphQL. See §3.1.

`Plan.live_state` is not on supervisor `_Meeting`; meeting `live_state` is `expect` on `MeetingBridge`.

### 3.1 `PlanRequester`

`@requester("plan")`, `@sub(["cpanel"], ["supervisor"])`: `read`, `create`, `update`, `delete`.

Every sub: `supervisor.can("PLAN", { sub, plan?, throwMode: true })`. Delete with any `Subscription` on `plan_id` throws `CANNOT_DELETE_USED`. Destroy uses `{ force: true }`.

Joi: `isPlan` for entity; `isMultiLang(joi)` for required name; `isMultiLang(joi, 4, true)` for optional description (null / both-empty → `null`; `.trim()` on ar/en). Prices `min(1)`. Limits optional integer `min(1)` or null.

`_Me.canDeletePlan(planId)` is a Me extra (`visualMode`) for hiding the form delete button. The requester remains the write authority.

This slice also changed cpanel `startTransaction()` calls from `"join_current"` to `startTransaction()` on `SupervisorRequester`, `CustomerRequester`, `PlatformSettingsRequester`, `WebsiteSettingsRequester`.

## 4) Subscriptions

Query: `subscriptions(filter: _SubscriptionFilter)`, `subscription(id: ID!)`, `subscriptionStats`.

`_SubscriptionFilter`: `search`, `status`. Search loads matching `Customer` rows (`name` / `email` iLike) then filters `customer_id IN (...)`. Empty match uses `id IN [0]` so the list is empty. Plan name is not searched. Cpanel list UI currently sends `search` only.

`_SubscriptionStats` extras: `total_count`, `active_count` (`status: "ACTIVE"`).

Nested `_Subscription.plan`, `_Subscription.customer`. Read-only in cpanel UI.

## 5) Meetings

Query: `meetings(filter: _MeetingFilter)`, `meeting(id: ID!)`, `meetingStats`.

`_MeetingFilter`: `search` (subject iLike), optional `status` (`_MeetingStatusValue`). The meetings directory UI sends `search` only. Home uses `status` via aliased root queries (`STARTED` / `WAITING_TO_START`).

`registerOrmAttrs.expect`: `["live_state"]` — live BLOB never leaves GraphQL.

`_MeetingStats` extras: `total_count`, `started_count` (`status: "STARTED"` only).

Nested: `_Meeting.organization` (`_Organization`), `_Meeting.chairperson` (`_Member`: `id`, `name`, `avatar_url`). `MemberBridge` is nested-only (no supervisor Member list root). `GetOneParent` includes `MeetingModel`.

Ops fields on `_Meeting` (type, datetime, notify_status, timestamps) as in SDL.

## 6) `Query.home` — `HomeBridge`

Same shape as supervisor `MeBridge`: `ident: "home"`, `ormModel: SupervisorModel`, `getRootOrmParent` → `"STATIC"`, `where: { id: supervisor.id }`. Dummy supervisor row so extras can load (`ignoreExtra` skips extras on many).

**Forbidden:** `static relations` to meetings (no Supervisor→Meeting association). Home lists of meetings use **root** `meetings(filter:)` with GraphQL aliases.

### 6.1 Count extras (`Int!`, prefixed so they do not collide with pagination `total_count`)

Customers: `customers_total_count`, `customers_created_today_count` (`countOfCreatedToday`).

Meetings: `meetings_total_count` plus per-status `meetings_draft_count`, `meetings_waiting_to_start_count`, `meetings_started_count`, `meetings_completed_count`, `meetings_canceled_count`.

Subscriptions: `subscriptions_total_count`, `subscriptions_active_count`, `subscriptions_expired_count`, `subscriptions_replaced_count`.

Plans: `plans_total_count`, `plans_active_count`, `plans_disabled_count`.

Directory `*Stats` roots stay for list headers. Home does not replace them; it may duplicate the same ORM counts under prefixed names.

### 6.2 `new_subscriptions_by_month: [_HomeMonthCount!]!`

Extra (not a relation). Calendar **January–December of the year of** `Subscription.THIS_MONTH_START()` (not a rolling window ending at the current month). Each bucket: `created_at >= monthStart AND created_at < nextMonth`. Missing months are `count: 0`.

`_HomeMonthCount`: `year_month` (`YYYY-MM`), `count`.

## 7) Authz / principal

`SupervisorSchema.getEnvContext` requires a supervisor row (current bootstrap `findByPk(1)`). Missing supervisor → `NOT_PERMIT`. All listed roots are platform-wide supervisor reads (no customer tenancy).

## 8) Codegen and mirrors

`backend/` `yarn generate-types` then `yarn type-check`. Command-copy `supervisor.graphql` + `gql-types/supervisor.ts` to `cpanel/src/types/gql/**`. Do not change website `customer` SDL for these surfaces.

## 9) Traceability

| Path | Role |
|---|---|
| `backend/src/app/gql/definitions/supervisor.graphql` | SDL |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated |
| `backend/src/app/gql/schemas/SupervisorSchema.ts` | Resolvers + `registeredBridges` |
| `backend/src/app/gql/bridges/supervisor/HomeBridge.ts` | Home extras |
| `backend/src/app/gql/bridges/supervisor/PlanBridge.ts` | Plan list/detail |
| `backend/src/app/gql/bridges/supervisor/PlanStatsBridge.ts` | Plan stats |
| `backend/src/app/gql/bridges/supervisor/SubscriptionBridge.ts` | Subscription list/detail |
| `backend/src/app/gql/bridges/supervisor/SubscriptionStatsBridge.ts` | Subscription stats |
| `backend/src/app/gql/bridges/supervisor/MeetingBridge.ts` | Meeting list/detail; `live_state` expect |
| `backend/src/app/gql/bridges/supervisor/MeetingStatsBridge.ts` | Meeting stats |
| `backend/src/app/gql/bridges/supervisor/MemberBridge.ts` | Nested chairperson |
| `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts` | `GetOneParent` includes Customer/Subscription/Organization for nests |
| `backend/src/app/gql/bridges/supervisor/MeBridge.ts` | `canDeletePlan` extra |
| `backend/src/app/orchestrator/requesters/PlanRequester.ts` | Plan writes |
| `backend/src/app/notify/events/OnSupervisorEvent.ts` | Socket `UPDATED` |

## Related

- `docs/platforms/cpanel/supervisor-home.md`
- `docs/platforms/cpanel/plan-management.md`
- `docs/platforms/cpanel/subscription-management.md`
- `docs/platforms/cpanel/meeting-management.md`
- `.cursor/rules/cpanel-supervisor-home.mdc`
- `.cursor/skills/gql-schema-bridge-generator/SKILL.md`
