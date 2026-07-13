# CPanel Platform Overview

## 1) Purpose

`cpanel/` is the Ejtmaa supervisor SSR frontend, served on backend mount `/cpanel`.

Platform docs describe the supervisor cpanel contract; the checked-in repository implements that contract.

Typed authed actor: **SUPERVISOR**. Visitor requester scope applies to login forms only.

## 2) Workspace relationship

| Platform | Role | Mount |
|---|---|---|
| `backend/` | API, requesters, ORM, GQL | `/website`, `/cpanel` |
| `website/` | Customer SSR | `/website` |
| `cpanel/` | Supervisor SSR | `/cpanel` |

Both frontends use `@my-ssr/web-core` + `@typescript/sys-core` with the same folder ownership model (`resources/`, `services/`, `ui/base/`, `ui/components/`, `ui/layouts/`, `ui/pages/`, `types/`).

## 3) Repository layout

| Path | Role |
|---|---|
| `src/client/*`, `src/server/*` | Browser and SSR entrypoints |
| `src/resources/configs/web-core.ts` | Runtime configuration authority |
| `src/resources/configs/routes.ts` | Route registry |
| `src/app/services/*` | Auth, router, boot, socket |
| `src/app/ui/base/*` | Framework infrastructure (`MyApp`, `MyPage`, hooks) |
| `src/app/ui/components/*` | Shared product UI (shell, tables, auth) |
| `src/app/ui/layouts/*` | `BasicLayout`, `MainLayout` |
| `src/app/ui/pages/*` | Route entry pages |
| `src/types/gql/**` | Local `base` + `supervisor` GQL mirrors |

## 4) Boot and auth

- Server boot calls `/cpanel/custom/start` and hydrates via `global.setServerStartData(...)`.
- Unauthenticated navigation redirects to `Login`.
- Authenticated users on `Login` redirect to `Home`.
- `UiMockup` is public for shell review.
- `Error` bypasses auth middleware once matched.

## 5) Route catalog

See `docs/platforms/cpanel/supervisor-admin-modules.md` for the authoritative module list.

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `MAIN` |
| `Customers` | `/customers` | `MAIN` |
| `Customer` | `/customer/:formType(show)/:id` | `MAIN` |
| `AccountSettings` | `/account-settings` | `MAIN` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `Error` | `/:error(404|500|403)` | `BASIC` |

## 6) Backend coupling

Supervisor GQL (`supervisor.graphql`):

- `me`, `notifications`
- `customers`, `customer(id)`
- `customerStats` (`total_count` for `HomeStatCard` on supervisor home)

Requesters on cpanel platform:

- `auth`, `supervisor`, `customer`, `website_settings`, `platform_settings`

Reads: `DATA_ADAPTERS.GQL` with supervisor schema.
Writes: `FORMS.SUPERVISOR.R`.

Socket namespace: `/supervisor`. Event: `OnUserEvent`.

## 7) UI foundation

Mandatory foundation:

- `ui/base/components/Utils.tsx`
- `resources/configs/theme.ts` (when present in cpanel checkout; navy `#0B2057`, orange `#EC6901`)
- `resources/configs/utils.ts`

See `docs/platforms/cpanel/ui-foundation.md`.

## 8) Related

- `docs/platforms/cpanel/README.md` -- full doc index
- `docs/platforms/cpanel/supervisor-admin-modules.md` -- module catalog
- `docs/platforms/cpanel/customer-management.md` -- customer list/detail
- `docs/invariants/cpanel.md` -- invariants
- `.cursor/rules/cpanel-platform-governance.mdc` -- governance rule
