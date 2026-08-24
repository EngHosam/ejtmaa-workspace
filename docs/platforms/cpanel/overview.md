# CPanel Platform Overview

## 1) Purpose

`cpanel/` is the Ejtmaa supervisor SSR frontend, served on backend mount `/cpanel`.

The checked-in frontend is the **supervisor workspace**: login, occupancy `Home` at `/`, dashboard `SupervisorHome` at `/supervisor`, read directories (customers, meetings, subscriptions), plan catalog writes, supervisor settings, and framework Error. Authed chrome is `SupervisorMainLayout` (Website `CustomerMainLayout` DNA). Shell identity is GQL `Query.me` (`useMe`). It shares the Website project's engineering DNA (`@my-ssr/web-core`, `@typescript/sys-core`, adapters, forms, Utils, GQL mirrors) without the Website customer/public product surface.

Typed authed actor: **SUPERVISOR**. Visitor requester scope applies to login forms only.

Backend supervisor contracts for customers, catalog, meetings, and `home` extras are consumed by cpanel. Notifications UI and an organizations directory are **not** implemented.

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
| `src/app/ui/components/*` | Shared reusable UI (shell, tables, auth, forms) |
| `src/app/ui/layouts/*` | `BasicLayout`, `SupervisorMainLayout` |
| `src/app/ui/pages/*` | Route entry pages |
| `src/types/gql/**` | Local `base` + `supervisor` GQL mirrors |

## 4) Boot and auth

- Server boot calls `GET /cpanel/custom/start` and hydrates via `global.setServerStartData(...)`.
- Unauthenticated navigation redirects to `Login`.
- Authenticated supervisors on `Login` redirect to `SupervisorHome` (`/supervisor`).
- `Home` occupies `/` (empty page) so the mount root is not unmatched; it is **not** a `publicRoutes` entry, so an unauthenticated visit redirects to `Login`.
- `SupervisorHome` renders `SUPERVISOR_MAIN` with real KPIs/charts from `Query.home` plus aliased meeting lists (`supervisor-home.md`). Directory headers still bind `Stats` to `*Stats` extras.
- `Error` bypasses auth middleware once matched.

## 5) Implemented route catalog

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `BASIC` (empty `MyPage`; occupies `/` so the mount root is not unmatched) |
| `SupervisorHome` | `/supervisor` | `SUPERVISOR_MAIN` |
| `SupervisorCustomers` | `/supervisor/customers` | `SUPERVISOR_MAIN` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | `SUPERVISOR_MAIN` |
| `SupervisorMeetings` | `/supervisor/meetings` | `SUPERVISOR_MAIN` |
| `SupervisorMeeting` | `/supervisor/meetings/:id` | `SUPERVISOR_MAIN` |
| `SupervisorPlans` | `/supervisor/plans` | `SUPERVISOR_MAIN` |
| `SupervisorPlanForm` | `/supervisor/plans/form` and `/supervisor/plans/form/:id` | `SUPERVISOR_MAIN` |
| `SupervisorSubscriptions` | `/supervisor/subscriptions` | `SUPERVISOR_MAIN` |
| `SupervisorSubscription` | `/supervisor/subscriptions/:id` | `SUPERVISOR_MAIN` |
| `SupervisorSettings` | `/supervisor/settings` | `SUPERVISOR_MAIN` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` |

`mustAuthedAs: ["SUPERVISOR"]` paths are built with `supervisorRouter` in `cpanel/src/resources/configs/routes.ts`. `Login`, `Home`, and `Error` use absolute paths. `publicRoutes` is `Login` only.

Identifies keep the `Supervisor` prefix (not `Customers` / `Customer`). There is no `SupervisorSupport` route.

## 6) Backend coupling

Supervisor GQL (`supervisor.graphql`) is mirrored under `cpanel/src/types/gql/`. Shell identity reads `Query.me` via `DATA_ADAPTERS.SUPERVISOR_ME`. Directories and home inherit `DATA_ADAPTERS.SUPERVISOR_GQL` (mount-private slots). See feature docs listed in `README.md`.

Requesters on the cpanel platform (backend): `auth` (visitor `supervisorLogin`), plus supervisor `customer`, `plan`, `platform_settings`, `supervisor`, `website_settings`. Login uses visitor `auth/supervisorLogin` only. Customers / meetings / subscriptions UI does not call write requesters.

Reads: `DATA_ADAPTERS.SUPERVISOR_ME` (`me { id name email }`) and `DATA_ADAPTERS.SUPERVISOR_GQL`.
Writes: `FORMS.SUPERVISOR.R("plan")`, `FORMS.SUPERVISOR.R("supervisor")`; login uses visitor auth form.

Socket namespace: `supervisor`. Events: `OnUserEvent` (`NEW_CUSTOMER` | `NEW_VENDOR`); `OnSupervisorEvent` (`UPDATED`, reloads `me`).

## 7) UI foundation

Mandatory foundation:

- `ui/base/components/Utils.tsx`
- `resources/configs/theme.ts`
- `resources/configs/utils.ts`

Arabic-only locale wiring (`locales: ["ar"]`). Copy lives in `src/resources/translations/ar.ts` only. `LanguageSwitch` is kept as a reusable component and is not mounted in chrome.

Local dev port: **3095**.

## 8) Related

- `docs/platforms/cpanel/flow-supervisor-shell.md` — supervisor workspace chrome
- `docs/platforms/cpanel/route-registry-contract.md` — `supervisorRouter` and occupancy `Home`
- `docs/platforms/cpanel/customer-management.md` — read-only customers directory
- `docs/platforms/cpanel/supervisor-home.md` — dashboard
- `docs/platforms/cpanel/supervisor-admin-modules.md` — implemented vs deferred
- `docs/invariants/cpanel.md` — invariants
- `.cursor/rules/cpanel-platform-governance.mdc` — governance rule
- `.cursor/rules/cpanel-supervisor-read-directory.mdc` — list/detail module invariants
- `.cursor/rules/cpanel-supervisor-home.mdc` — home dashboard invariants
