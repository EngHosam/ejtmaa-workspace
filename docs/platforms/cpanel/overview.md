# CPanel Platform Overview

## 1) Purpose

`cpanel/` is the Ejtmaa supervisor SSR frontend, served on backend mount `/cpanel`.

The checked-in frontend is a **supervisor bootstrap plus a read-only customers directory**: login, occupancy `Home` at `/`, empty authenticated `SupervisorHome` at `/supervisor`, `SupervisorCustomers` / `SupervisorCustomer`, and framework Error. Authed chrome is `SupervisorMainLayout` (Website `CustomerMainLayout` DNA). Shell identity is GQL `Query.me` (`useMe`). It shares the Website project's engineering DNA (`@my-ssr/web-core`, `@typescript/sys-core`, adapters, forms, Utils, GQL mirrors) without the Website customer/public product surface.

Typed authed actor: **SUPERVISOR**. Visitor requester scope applies to login forms only.

Backend supervisor contracts for customers/stats are consumed by the cpanel customers UI. Account settings and home KPI widgets are **not** implemented.

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
- `SupervisorHome` renders `SUPERVISOR_MAIN` with **no dashboard widgets**. Customer aggregates appear on the customers list header (`Stats` ← `customerStats`), not on home.
- `Error` bypasses auth middleware once matched.

## 5) Implemented route catalog

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `BASIC` (empty `MyPage`; occupies `/` so the mount root is not unmatched) |
| `SupervisorHome` | `/supervisor` | `SUPERVISOR_MAIN` |
| `SupervisorCustomers` | `/supervisor/customers` | `SUPERVISOR_MAIN` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | `SUPERVISOR_MAIN` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` |

`mustAuthedAs: ["SUPERVISOR"]` paths are built with `supervisorRouter` in `cpanel/src/resources/configs/routes.ts`. `Login`, `Home`, and `Error` use absolute paths. `publicRoutes` is `Login` only.

There is no `AccountSettings` route. Identifies are `SupervisorCustomers` / `SupervisorCustomer` (not `Customers` / `Customer`).

## 6) Backend coupling

Supervisor GQL (`supervisor.graphql`) is mirrored under `cpanel/src/types/gql/`. Shell identity reads `Query.me` via `DATA_ADAPTERS.SUPERVISOR_ME`. Customers list/detail inherit `DATA_ADAPTERS.SUPERVISOR_GQL` (mount-private slots). See `customer-management.md`.

Requesters on the cpanel platform (backend): `auth` (visitor `supervisorLogin`), plus supervisor `customer`, `platform_settings`, `supervisor`, `website_settings` read|update. Login uses visitor `auth/supervisorLogin` only. The customers UI does not call customer requesters.

Reads: `DATA_ADAPTERS.SUPERVISOR_ME` (`me { id name email }`) and `DATA_ADAPTERS.SUPERVISOR_GQL` for list/detail modules (`API.DATA_ADAPTERS.SUPERVISOR.GQL`).
Writes: `FORMS.SUPERVISOR.R` (foundation; login uses visitor auth form).

Socket namespace: `supervisor`. Event: `OnUserEvent` with `type: "NEW_CUSTOMER" | "NEW_VENDOR"`.

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
- `docs/platforms/cpanel/supervisor-admin-modules.md` — implemented vs deferred
- `docs/invariants/cpanel.md` — invariants
- `.cursor/rules/cpanel-platform-governance.mdc` — governance rule
- `.cursor/rules/cpanel-supervisor-read-directory.mdc` — list/detail module invariants
