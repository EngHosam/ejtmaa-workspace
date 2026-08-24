# CPanel Repository Inventory

## Purpose

Inventory of project-owned paths under `cpanel/` for the current supervisor checkout (workspace modules, not bootstrap-only).

This is the live checkout inventory, not a future product catalog.

## Runtime entry

| Path | Role |
|---|---|
| `src/client/index.ts`, `src/client/worker.ts` | Browser bootstrap |
| `src/server/index.ts`, `src/server/worker.ts` | SSR bootstrap |
| `src/resources/configs/web-core.ts` | Runtime configuration authority |
| `src/resources/configs/routes.ts` | Route registry |
| `src/resources/configs/store/data-adapters.ts` | `SUPERVISOR_ME` + `SUPERVISOR_GQL` |
| `src/resources/configs/urls.ts` | `ME_URL` / `BASE_URL` / `SOCKET_URL` (port 3095, mount `/cpanel`) |

## Services

| Path | Role |
|---|---|
| `src/app/services/index.ts` | Server/client boot phases |
| `src/app/services/auth.ts` | `SUPERVISOR` auth helpers + `loadCurrentSupervisor` |
| `src/app/services/router.ts` | Route guards and redirects |

## UI foundation

| Path | Role |
|---|---|
| `src/app/ui/base/components/Utils.tsx` | Layout/styling primitives |
| `src/resources/configs/theme.ts` | Brand tokens |
| `src/app/ui/base/core/MyApp.tsx` | Layout resolution (`BASIC` / `SUPERVISOR_MAIN`) |
| `src/app/ui/base/core/MyHtml.tsx` | Document shell |
| `src/app/ui/base/core/MyPage.tsx` | Route lifecycle |

## Shared shell

| Path | Role |
|---|---|
| `src/app/ui/layouts/BasicLayout.tsx` | Auth/error wrapper |
| `src/app/ui/layouts/SupervisorMainLayout.tsx` | Authed supervisor shell |
| `src/app/ui/components/supervisor/*` | Header, drawer, footer, sub-header, home, `hooks/useMe`, customers/meetings/plans/subscriptions/settings |
| `src/app/ui/components/Stats.tsx` | Shared KPI tile grid (`href` / `note`) |
| `src/app/ui/components/PageStateLane.tsx` | Load / empty / fail overlay host (detail) |
| `src/app/ui/components/FormStateLane.tsx` | Alias of `PageStateLane` (forms) |
| `src/app/ui/components/ResultLane.tsx` | Directory card grid + empty/fail |
| `src/app/ui/components/StatusChip.tsx` | Meeting / org / subscription status pills |
| `src/app/ui/components/DataTable.tsx` | List table primitive (reusable; unused by directories — ResultLane) |
| `src/app/ui/components/IdentityAvatar.tsx` | Remapped reusable avatar (from Website `components/customer/`) |
| `src/app/ui/components/auth/*` | Login shell DNA |
| `src/app/ui/components/form/FormMultiLangField.tsx` | Nested ar/en form field |
| `src/app/ui/components/form/*` | Remaining form field DNA |
| `src/app/ui/components/modals/*` | Entity / datetime / confirm modals |
| `src/app/ui/components/LanguageSwitch.tsx` | Locale switcher component (unmounted; Arabic-only wiring) |

## Route pages

| Path | Route |
|---|---|
| `src/app/ui/pages/Login.tsx` | `/login` |
| `src/app/ui/pages/Home.tsx` | `/` (empty `MyPage`) |
| `src/app/ui/pages/supervisor/SupervisorHome.tsx` | `/supervisor` (dashboard) |
| `src/app/ui/pages/supervisor/SupervisorCustomers.tsx` | `/supervisor/customers` |
| `src/app/ui/pages/supervisor/SupervisorCustomer.tsx` | `/supervisor/customers/:id` |
| `src/app/ui/pages/supervisor/SupervisorMeetings.tsx` | `/supervisor/meetings` |
| `src/app/ui/pages/supervisor/SupervisorMeeting.tsx` | `/supervisor/meetings/:id` |
| `src/app/ui/pages/supervisor/SupervisorPlans.tsx` | `/supervisor/plans` |
| `src/app/ui/pages/supervisor/SupervisorPlanForm.tsx` | `/supervisor/plans/form` (+ `/:id`) |
| `src/app/ui/pages/supervisor/SupervisorSubscriptions.tsx` | `/supervisor/subscriptions` |
| `src/app/ui/pages/supervisor/SupervisorSubscription.tsx` | `/supervisor/subscriptions/:id` |
| `src/app/ui/pages/supervisor/SupervisorSettings.tsx` | `/supervisor/settings` |
| `src/app/ui/pages/Error.tsx` | `/:error(404\|500\|403)` |

Feature UI lives under `components/supervisor/<module>/`. Exhaustive path table: `supervisor-workspace-change-set.md`.

## Types and GQL mirrors

| Path | Role |
|---|---|
| `src/types/requesters/requesters.cpanel.ts` | Supervisor requester map |
| `src/types/gql/definitions/base.graphql` | Mirrored base SDL |
| `src/types/gql/definitions/supervisor.graphql` | Mirrored supervisor SDL |
| `src/types/gql/gql-types/base.ts` | Generated base types |
| `src/types/gql/gql-types/supervisor.ts` | Generated supervisor types |
| `src/types/events.ts` | `OnUserEvent` (`NEW_CUSTOMER` \| `NEW_VENDOR`); `OnSupervisorEvent` (`UPDATED`) |

## Identity

| Item | Value |
|---|---|
| Package name | `ejtmaa-cpanel` |
| Dev port | `3095` |
| Nested git | `cpanel/` is its own repository |
