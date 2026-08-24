# CPanel Supervisor Shell

Supervisor workspace chrome on `cpanel/`, matching Website `CustomerMainLayout` DNA without customer product modules.

## 1) Layout

- Identify: `SUPERVISOR_MAIN`
- File: `cpanel/src/app/ui/layouts/SupervisorMainLayout.tsx`
- `MyApp.getLayout()` maps `SUPERVISOR_MAIN` → `SupervisorMainLayout`
- `BASIC` remains `Login`, occupancy `Home`, and `Error`

Composition order (same as Website customer shell):

1. `SupervisorDrawer` (`open` / `onClose`) — layout sibling
2. `SupervisorHeader` (`onMenuClick` toggles drawer)
3. Optional fixed `SupervisorSubHeader` when `routes[identify]?.breadcrumb` is set
4. Page content (`Col` `minH 100vh` with header / sub-header offset so children can `flx_1`)
5. `SupervisorFooter`

There is no `MainLayout`, `UiMockup`, or card-style `Header` / `Drawer` / `Footer` in this checkout.

## 2) Routes

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `BASIC` (empty page; occupies `/`) |
| `SupervisorHome` | `/supervisor` | `SUPERVISOR_MAIN` |
| `SupervisorCustomers` | `/supervisor/customers` | `SUPERVISOR_MAIN` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | `SUPERVISOR_MAIN` |
| `SupervisorMeetings` | `/supervisor/meetings` | `SUPERVISOR_MAIN` |
| `SupervisorMeeting` | `/supervisor/meetings/:id` | `SUPERVISOR_MAIN` |
| `SupervisorPlans` | `/supervisor/plans` | `SUPERVISOR_MAIN` |
| `SupervisorPlanForm` | create `/supervisor/plans/form`; update `/supervisor/plans/form/:id` | `SUPERVISOR_MAIN` |
| `SupervisorSubscriptions` | `/supervisor/subscriptions` | `SUPERVISOR_MAIN` |
| `SupervisorSubscription` | `/supervisor/subscriptions/:id` | `SUPERVISOR_MAIN` |
| `SupervisorSettings` | `/supervisor/settings` | `SUPERVISOR_MAIN` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` |

`mustAuthedAs: ["SUPERVISOR"]` paths go through `supervisorRouter` in `routes.ts` (`supervisorRouter("")` → `/supervisor`). `Login`, `Home`, and `Error` stay absolute. `publicRoutes` is `Login` only.

`getMyHomeIdentify` returns `SupervisorHome`. Authed supervisors on `Login` redirect there. `Home` is registered at `/` but is not in `publicRoutes`; unauthenticated visits redirect to `Login`.

`SupervisorHome` is a thin `MyPage` → `SupervisorHomeScreen` (dashboard). Contract: `supervisor-home.md`.

## 3) Header

`SupervisorHeader`: fixed bar, blur, 2px accent rail. Start cluster: menu (`DrawerMenuIcon`) → desktop-only `Logo` (`hideAt maxW md`, click → `SupervisorHome`) → identity (`IdentityAvatar` + greeting from `useMe().me.name`). Trailing: notifications bell, disabled until a notifications route exists.

## 3.1) Supervisor `me` — `useMe` + `SUPERVISOR_ME`

- Adapter id: `DATA_ADAPTERS.SUPERVISOR_ME` in `cpanel/src/resources/configs/store/data-adapters.ts`, API `API.DATA_ADAPTERS.SUPERVISOR.GQL` (`GET /cpanel/data_adapters/supervisor/gql`).
- Hook: `cpanel/src/app/ui/components/supervisor/hooks/useMe.tsx`.
  - Default: `useAdapter` + core query `me { id name email }` (`LOAD_ON_MOUNT`, or `FORCE_RELOAD_ON_MOUNT` when `updateOnEnter`).
  - With `data.query`: `useShallowAdapter` keyed by `md5(data.query)`, inherits `SUPERVISOR_ME`.
- `_Me` in supervisor SDL has **no** `avatar_url`. Chrome uses `IdentityAvatar` initials/icon from `name` only. Auth `start` still hydrates `auth.supervisor.permissions` only; profile identity is this GQL adapter, not the auth reducer.
- SSR: `auth.loadCurrentSupervisor(myInstance)` in `cpanel/src/app/services/auth.ts` (no-op unless `isAuthedAs(..., ["SUPERVISOR"])`) calls `LoadCurrentSupervisor`; invoked from `boot.server` after start + permissions.
- Socket: `useSocket({ registerTo: "OnSupervisorEvent", ... })` → `mLoad({reload: true})` (same pattern as website `OnCustomerEvent`). `OnUserEvent` (`NEW_CUSTOMER` | `NEW_VENDOR`) does not reload `me`.

## 4) Drawer

Portaled grid (Website customer drawer DNA). Order: Home (`HomeMark` `tone="onPrimary"`) → Customers → Meetings → Plans → Subscriptions → Support (rendered, `available=false`, no route) → Settings. First-class tiles use `alwaysAvailable`. Utility: theme switch (no language switch; Arabic-only) + logout.

## 5) Sub-header

`SupervisorSubHeader` + `useBreadcrumbs({ rootIdentify: "SupervisorHome" })`. Workspace routes register `breadcrumb` in `routes.ts`.

## 6) i18n

`ui.layouts.supervisorMainLayout` and `ui.pages.supervisor` (home, customers, meetings, plans, subscriptions, settings) in `cpanel/src/resources/translations/ar.ts`. There is no `en.ts` (Arabic-only `locales: ["ar"]`).

## 7) Change-set inventory (cpanel sources)

| Path | Role | Documented |
|---|---|---|
| `src/app/ui/layouts/SupervisorMainLayout.tsx` | Authed shell tree; content `Col` for flex fill | §1 |
| `src/app/ui/components/supervisor/SupervisorHeader.tsx` | Fixed header | §3 |
| `src/app/ui/components/supervisor/SupervisorDrawer.tsx` | Portaled drawer (Home through Settings) | §4 |
| `src/app/ui/components/supervisor/SupervisorFooter.tsx` | Footer + MicrobandCredit | §1 |
| `src/app/ui/components/supervisor/SupervisorSubHeader.tsx` | Breadcrumb bar | §5 |
| `src/app/ui/pages/supervisor/SupervisorCustomers.tsx` | Customers list page | `customer-management.md` |
| `src/app/ui/pages/supervisor/SupervisorCustomer.tsx` | Customer detail page | `customer-management.md` |
| `src/app/ui/components/supervisor/home/SupervisorHomeScreen.tsx` | Dashboard composition | `supervisor-home.md` |
| `src/app/ui/components/supervisor/hooks/useMe.tsx` | GQL `me` adapter hook | §3.1 |
| `src/app/ui/components/supervisor/graphql.config.yml` | Editor schema `base` + `supervisor` | `graphql-mirror-and-tooling.md` |
| `src/app/ui/pages/supervisor/SupervisorHome.tsx` | Route page | `supervisor-home.md` |
| `src/app/ui/pages/supervisor/graphql.config.yml` | Editor schema `base` + `supervisor` | `graphql-mirror-and-tooling.md` |
| `src/app/ui/pages/Home.tsx` | `/` occupancy | `route-registry-contract.md` |
| `src/app/ui/pages/Error.tsx` | CTA `SupervisorHome` | `error-route-and-guard.md` |
| `src/app/ui/pages/graphql.config.yml` | Generic pages: `base` only | `graphql-mirror-and-tooling.md` |
| `src/app/ui/components/graphql.config.yml` | Generic components: `base` only | `graphql-mirror-and-tooling.md` |
| `src/app/ui/components/auth/AuthPageShell.tsx` | Logo href `SupervisorHome` | `login-runtime-and-feedback.md` |
| `src/app/ui/base/core/MyApp.tsx` | `SUPERVISOR_MAIN` layout map | §1 |
| `src/types/extends/global.ts` | `Layout = "BASIC" \| "SUPERVISOR_MAIN"` | this platform |
| `src/resources/configs/routes.ts` | Registry | `route-registry-contract.md` |
| `src/resources/configs/store/data-adapters.ts` | `SUPERVISOR_ME` | §3.1 |
| `src/resources/translations/ar.ts` | Shell + home + customers + login copy | §6 |
| `src/app/services/auth.ts` | `loadCurrentSupervisor` | §3.1 |
| `src/app/services/index.ts` | Boot hydrate `me` | §3.1 |
| `src/app/services/router.ts` | Guards | `route-registry-contract.md` |
| Deleted: `MainLayout.tsx`, `main-layout/drawer.ts`, `Header.tsx`, `Footer.tsx`, `Drawer.tsx`, `UiMockup.tsx`, `en.ts` | Removed Website leftover shell | §1 |

Generated `cpanel/lib/tsconfig.tsbuildinfo` is a `tsc` artifact, not a product contract.

## Related

- `docs/platforms/cpanel/route-registry-contract.md`
- `docs/platforms/cpanel/customer-management.md`
- `docs/platforms/cpanel/overview.md`
