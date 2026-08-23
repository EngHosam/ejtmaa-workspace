# CPanel Repository Inventory

## Purpose

Inventory of project-owned paths under `cpanel/` for the current supervisor bootstrap.

This is the live checkout inventory, not a future product catalog.

## Runtime entry

| Path | Role |
|---|---|
| `src/client/index.ts`, `src/client/worker.ts` | Browser bootstrap |
| `src/server/index.ts`, `src/server/worker.ts` | SSR bootstrap |
| `src/resources/configs/web-core.ts` | Runtime configuration authority |
| `src/resources/configs/routes.ts` | Route registry |
| `src/resources/configs/urls.ts` | `ME_URL` / `BASE_URL` / `SOCKET_URL` (port 3095, mount `/cpanel`) |

## Services

| Path | Role |
|---|---|
| `src/app/services/index.ts` | Server/client boot phases |
| `src/app/services/auth.ts` | `SUPERVISOR` auth helpers |
| `src/app/services/router.ts` | Route guards and redirects |

## UI foundation

| Path | Role |
|---|---|
| `src/app/ui/base/components/Utils.tsx` | Layout/styling primitives |
| `src/resources/configs/theme.ts` | Brand tokens |
| `src/app/ui/base/core/MyApp.tsx` | Layout resolution (`BASIC` / `MAIN`) |
| `src/app/ui/base/core/MyHtml.tsx` | Document shell |
| `src/app/ui/base/core/MyPage.tsx` | Route lifecycle |

## Shared shell

| Path | Role |
|---|---|
| `src/app/ui/layouts/BasicLayout.tsx` | Auth/error wrapper |
| `src/app/ui/layouts/MainLayout.tsx` | Authed shell (header, drawer, footer) |
| `src/app/ui/layouts/main-layout/drawer.ts` | Drawer sections: dashboard + logout |
| `src/app/ui/components/Header.tsx` | Shell header |
| `src/app/ui/components/Drawer.tsx` | Navigation drawer |
| `src/app/ui/components/Footer.tsx` | Shell footer |
| `src/app/ui/components/DataTable.tsx` | List table primitive (reusable; unused by current pages except UiMockup) |
| `src/app/ui/components/IdentityAvatar.tsx` | Remapped reusable avatar (from Website `components/customer/`) |
| `src/app/ui/components/auth/*` | Login shell DNA |
| `src/app/ui/components/form/*` | Form field DNA |
| `src/app/ui/components/modals/*` | Entity / datetime / confirm modals |
| `src/app/ui/components/LanguageSwitch.tsx` | Locale switcher component (unmounted; Arabic-only wiring) |

## Route pages

| Path | Route |
|---|---|
| `src/app/ui/pages/Login.tsx` | `/login` |
| `src/app/ui/pages/Home.tsx` | `/` (empty `Main()`) |
| `src/app/ui/pages/UiMockup.tsx` | `/ui-mockup` |
| `src/app/ui/pages/Error.tsx` | `/:error(404\|500\|403)` |

There are no `Customers.tsx`, `Customer.tsx`, `AccountSettings.tsx`, or `home/` KPI components in this checkout.

## Types and GQL mirrors

| Path | Role |
|---|---|
| `src/types/requesters/requesters.cpanel.ts` | Supervisor requester map |
| `src/types/gql/definitions/base.graphql` | Mirrored base SDL |
| `src/types/gql/definitions/supervisor.graphql` | Mirrored supervisor SDL |
| `src/types/gql/gql-types/base.ts` | Generated base types |
| `src/types/gql/gql-types/supervisor.ts` | Generated supervisor types |
| `src/types/socket/events.ts` | `OnUserEvent` / `NEW_CUSTOMER` \| `NEW_VENDOR` |

## Identity

| Item | Value |
|---|---|
| Package name | `ejtmaa-cpanel` |
| Dev port | `3095` |
| Nested git | `cpanel/` is its own repository |
