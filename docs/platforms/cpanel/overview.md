# CPanel Platform Overview

## 1) Purpose

`cpanel/` is the Ejtmaa supervisor SSR frontend, served on backend mount `/cpanel`.

The checked-in frontend is an **intentionally minimal bootstrap**: supervisor login, an empty authenticated Home, framework Error, and the `UiMockup` engineering review route. It shares the Website project's engineering DNA (`@my-ssr/web-core`, `@typescript/sys-core`, adapters, forms, Utils, GQL mirrors) without the Website customer/public product surface.

Typed authed actor: **SUPERVISOR**. Visitor requester scope applies to login forms only.

Backend supervisor contracts (customer list/detail/stats, account settings, and related GQL) already exist. They are **not** implemented as CPanel frontend modules in this bootstrap.

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
| `src/app/ui/layouts/*` | `BasicLayout`, `MainLayout` |
| `src/app/ui/pages/*` | Route entry pages |
| `src/types/gql/**` | Local `base` + `supervisor` GQL mirrors |

## 4) Boot and auth

- Server boot calls `GET /cpanel/custom/start` and hydrates via `global.setServerStartData(...)`.
- Unauthenticated navigation redirects to `Login`.
- Authenticated supervisors on `Login` redirect to `Home`.
- `Home` renders the MAIN shell with **no dashboard widgets**.
- `UiMockup` is a technical review route (`mustAuthedAs: ["SUPERVISOR"]`).
- `Error` bypasses auth middleware once matched.

## 5) Implemented route catalog

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `MAIN` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` |

There are no `Customers`, `Customer`, or `AccountSettings` routes in the current frontend.

## 6) Backend coupling

Supervisor GQL (`supervisor.graphql`) is mirrored under `cpanel/src/types/gql/` for future reads. Current pages do not query customer list/stats.

Requesters on the cpanel platform (backend): `auth` (visitor `supervisorLogin`), plus supervisor `customer`, `platform_settings`, `supervisor`, `website_settings` read|update. The bootstrap login uses visitor `auth/supervisorLogin` only.

Reads: `DATA_ADAPTERS.GQL` with supervisor schema (foundation; unused by current pages).
Writes: `FORMS.SUPERVISOR.R` (foundation; login uses visitor auth form).

Socket namespace: `supervisor`. Event: `OnUserEvent` with `type: "NEW_CUSTOMER" | "NEW_VENDOR"`.

## 7) UI foundation

Mandatory foundation:

- `ui/base/components/Utils.tsx`
- `resources/configs/theme.ts`
- `resources/configs/utils.ts`

Arabic-only locale wiring (`locales: ["ar"]`). `en.ts` is retained unused. `LanguageSwitch` is kept as a reusable component and is not mounted in chrome.

Local dev port: **3095**.

## 8) Related

- `docs/platforms/cpanel/README.md` — doc index
- `docs/platforms/cpanel/supervisor-admin-modules.md` — implemented vs deferred
- `docs/invariants/cpanel.md` — invariants
- `.cursor/rules/cpanel-platform-governance.mdc` — governance rule
