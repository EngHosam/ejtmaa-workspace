# CPanel Component and Folder Structure

## Purpose

This document defines ownership of the `cpanel/` file tree.
It exists so future work can expand the control panel without mixing bootstrap code, feature code, design-system code, and backend wiring into the same folders.

The control panel is an SSR web platform built around `@my-ssr/web-core`.

`website/` uses the same SSR scaffold family; organizational lessons transfer across web platforms.

However, both platforms intentionally share the same **organizational mental model** in several places:
- one UI foundation layer,
- one infrastructure layer,
- one data-adapter layer,
- one form/requester layer,
- one translation source,
- one place for route-owned screens.

Important scope rule:
- this document defines **ownership and boundaries**,
- it must not be read as proof that every reserved layer already contains finished feature implementations.

## 1) Top-level ownership

| Path | Ownership |
|------|-----------|
| `cpanel/src/client/` | Browser bootstrap and client worker entrypoints only. No feature business logic should start here. |
| `cpanel/src/server/` | SSR/bootstrap and server worker entrypoints only. No feature business logic should start here. |
| `cpanel/src/resources/` | Runtime resources and configuration authority: routes, store, axios, theme, translations, shell styles. |
| `cpanel/src/app/` | Cpanel-facing code: services, helpers, UI foundation, shared UI, layouts, and pages. |
| `cpanel/src/types/` | Project-owned type surfaces that should stay independent from concrete page files. |
| `cpanel/eng-hosam/` | Private framework/customer libraries. These are dependencies and reference material, not the place for Ejtmaa business implementation. |

## 2) `src/resources/` ? platform configuration and resource authority

`src/resources/` is the control-panel equivalent of the platform runtime registry.
This folder should contain **configuration and resource ownership**, not feature-specific page logic.

### 2.1 `src/resources/configs/`

This is the main configuration authority for the platform.

Examples and ownership:

| Path | Responsibility |
|------|----------------|
| `configs/web-core.ts` | Registers app shell, store, axios, routes, and localization. Route middleware timing is not owned here in the implementation. |
| `configs/routes.ts` | Declares page identifies and path mapping. |
| `configs/store.ts` | Wires reducers and preset reducers (`data_adapters`, `forms`, `modals`). |
| `configs/store/data-adapters.ts` | Shared adapter identifiers and default/init adapter props. |
| `configs/store/forms.ts` | Shared form identifiers and default/init form props. |
| `configs/store/reduces/*` | Redux reducer registry and reducer implementations. |
| `configs/axios.ts` and `configs/axios/api.ts` | HTTP client setup and backend `/cpanel` endpoint mapping. |
| `configs/urls.ts` | Environment-aware frontend/backend/socket base URL resolution. |
| `configs/theme.ts` | Base colors, theme map, breakpoints, dimensions, z-index (when present in cpanel checkout). |
| `configs/utils.ts` | Shared UI semantic helpers such as text variants, default text color, divider color, and container dimensions. |
| `configs/client.ts` and `configs/server.ts` | Client/server runtime funnel setup. |
| `configs/socket.ts` and `configs/socket/events.ts` | Socket connection/event configuration, including mirrored supervisor-facing backend events. |

Rule:
- if a value is a **platform-level runtime contract**, it belongs in `resources/configs/`.
- if a value is only relevant to one page or one business feature, it should **not** be promoted into `resources/configs/` prematurely.

### 2.2 `src/resources/translations/`

This folder owns user-facing copy and translation dictionaries.
Arabic is the active locale; this remains the ownership boundary for additional locales.

Rule:
- visible copy belongs here,
- page files and shared components should consume translation output, not become the source of truth for translated strings.

### 2.3 `src/resources/emotion/styles/`

This folder owns global Emotion styles and shell-level styling resources.

Rule:
- global document/app-shell styling belongs here,
- local component styling should stay with the component and be composed through `Utils`,
- page-specific business UI styling should not become a global stylesheet.

## 3) `src/app/services/` ? cpanel behavior services

`src/app/services/` owns control-panel behavior that is broader than one component but narrower than platform bootstrap.

Services show the intended split:

| Path | Responsibility |
|------|----------------|
| `services/index.ts` | Boot orchestration for server/client phases. |
| `services/router.ts` | Route access and redirect behavior; the pure middleware logic lives here, while invocation happens from `MyPage` lifecycle hooks. |
| `services/auth.ts` | Auth helpers, token cookie handling, role checks, logout/login behavior. |
| `services/global.ts` | Startup/global state shaping helpers. |
| `services/socket.ts` | Socket boot helper. |
| `services/pre-messages.ts` | Pre-message replay behavior. |
| `services/logger.ts` | Logging integration helper. |

Rule:
- services may coordinate between store, router, auth, socket, and transport,
- services should **not** own page markup or view composition,
- services should **not** bypass forms/adapters when a stateful boundary already exists for the same concern.

## 4) `src/app/ui/` ? UI ownership tree

This is the most important structural layer for future cpanel growth.
It must remain internally split by **foundation**, **shared project UI**, **layout**, and **route pages**.

### 4.1 `src/app/ui/base/` ? infrastructure UI foundation

`base/` is the infrastructure UI layer.
It is where the project binds its own UI contracts to the underlying shared libraries from `eng-hosam/`, `@my-ssr/web-core`, `@typescript/sys-core`, Emotion, router, Redux, and the platform store.

It is the closest equivalent to `website/src/components/base/`, but adapted to the SSR web stack.

#### `base/core/`

Core shell classes and runtime UI hosts:

| Path | Responsibility |
|------|----------------|
| `base/core/MyApp.tsx` | App shell composition, layout selection, global providers, and shell-level rendering. |
| `base/core/MyHtml.tsx` | HTML document shell, fonts, favicon/manifest links, and document direction. |
| `base/core/MyPage.tsx` | Shared page lifecycle wrapper and memoized page host; also applies router middleware during server render, client hydrate, and client navigation. |

Rule:
- do not place feature business UI here,
- this layer owns framework integration and page-host behavior only.

#### `base/components/`

Reusable low-level primitives and infrastructure components:

| Path | Responsibility |
|------|----------------|
| `base/components/Utils.tsx` | Primary UI primitive system for the web control panel. |
| `base/components/Adapter.tsx` | Component-level data-adapter boundary. |
| `base/components/Form.tsx` | Component-level form boundary. |
| `base/components/ShallowAdapter.tsx` | Injected adapter boundary wrapper. |
| `base/components/ShallowForm.tsx` | Injected form boundary wrapper. |
| `base/components/Injector.tsx` | Low-level injector plumbing. |
| `base/components/ModalsManager.tsx` / `ModalBase.tsx` | Modal host/infrastructure. |
| `base/components/Sensor.tsx` | Low-level infrastructure helper surface. |

Rule:
- `base/components/` exists to bridge the project with framework primitives and shared state infrastructure,
- it should remain generic and cross-feature,
- business cards, tables, dashboards, filters, and page sections do **not** belong here.

#### `base/hooks/`

Reusable framework-facing hooks:

Examples:
- router hooks,
- translator hooks,
- axios hooks,
- adapter/form hooks,
- injector hooks,
- socket hooks,
- theme-manager hooks,
- cookies/local storage helpers.

Rule:
- hooks in `base/hooks/` are infrastructure hooks,
- they should expose reusable mechanics, not one-screen business policies,
- if a hook becomes domain-specific later, it should move to a domain/shared feature folder outside `base/`.

### 4.2 `src/app/ui/components/` ? project shared UI

`components/` is for shared project UI that belongs to the control panel itself, but is **not** infrastructure/base.

Examples:

| Path | Responsibility |
|------|----------------|
| `components/Loadable.tsx` | Project-visible loading surface with inline/overlay Lottie behavior. |
| `components/GlobalLoadable.tsx` | App-level loading overlay tied to global busy state. |
| `components/Toast.tsx` | Shared cpanel toast rendering surface. |
| `components/auth/*` | Shared auth-page product UI (`AuthPageShell`, `AuthTextField`) built on the shared form primitives, with `AuthPageShell` owning auth-card width/max-width behavior instead of the route page. |
| `components/ThemeModeSwitch.tsx` | Shared theme-mode toggle for shell/layout review and future reuse. |
| `components/Header.tsx` / `Drawer.tsx` / `Footer.tsx` | Shared shell composition components that belong to product UI, not `base/`. |
| `components/Logo.tsx` | Shared light/dark logo switcher for shell branding surfaces. |
| `components/StatusBadge.tsx` | Shared reusable compact state badge for status-like labels across modules. |
| `components/DataTable.tsx` | Shared product table surface with toolbar/pagination review contract. |
| `components/form/*` | Shared project form primitives (`FormInputWrapper`, `FormTextField`, `FormActionButton`). |
| `components/customer/*` | Customer-module shared UI (`CustomersTable`, `CustomerStatsSection`, `CustomerIdentityCell`) reused across supervisor customer routes. |
| `components/home/*` | `HomeStatCard` for the `Home` route (`customerStats.total_count`). |

Rule:
- if a component is part of the **project UI language** rather than framework plumbing, it belongs here,
- this is where future admin widgets, tables, empty states, page headers, shell controls, cards, and reusable module sections should live,
- do not put those directly into `base/`.
- do not fragment one module into file-per-private-helper when the helper is only used inside one section; prefer section-level extraction and keep strictly local helpers inline in the owning file.
- local `graphql.config.yml` is allowed here only when `components/` contains GraphQL-aware UI/hooks and must point at the mirrored cpanel schema under `src/types/gql/`.

### 4.3 `src/app/ui/layouts/` ? shell layouts

Layouts define page shells and section-level outer composition.

State:
- `BasicLayout.tsx` exists as the minimal auth/error-like wrapper.
- `MainLayout.tsx` ? shared shell/page layout that composes `Header`, `Drawer`, and `Footer`.
- `layouts/main-layout/drawer.ts` ? layout-owned helper module for the `MAIN` shell navigation structure.

Rule:
- route pages render inside layouts selected by `MyApp`,
- shared shell composition belongs in `layouts/`,
- layout-specific helper modules should stay colocated under the owning layout folder/subfolder rather than being promoted into a generic component file prematurely,
- page-specific content should not be duplicated in every route when it can live in a reusable layout.

### 4.4 `src/app/ui/pages/` ? route-owned pages

`pages/` owns page entry components bound directly to route identifiers.
Supervisor pages:
- `Login` ? requester-backed shallow form and shared auth components.
- `Home` ? `HomeStatCard` bound to `customerStats.total_count`.
- `Customers` ? customer list with route-state-backed table.
- `Customer` ? single-record form (`customer.read` / `customer.update`).
- `AccountSettings` ? supervisor password change.
- `UiMockup` ? visual review page on the real layout.
- `Error` ? branded fallback page.

Rule:
- pages orchestrate route-level behavior and compose shared UI,
- they should stay thin and avoid owning deep reusable UI primitives,
- if a visual section is reusable or large, extract it into `components/` rather than bloating the page file,
- if a section is reusable only inside one route/module family, prefer one section-level component file rather than many tiny helper files,
- pages should consume hooks/services/forms/adapters; they should not reimplement those mechanisms inline.
- local `graphql.config.yml` is allowed here only when `pages/` contains GraphQL-aware route code and must point at the mirrored cpanel schema under `src/types/gql/`.

## 5) `src/types/` ? project-owned type contracts

This folder owns shared type surfaces that should not be trapped inside individual page files.

Groups:
- `types/requesters/requesters.cpanel.ts`
- `types/extends/`
- `types/events.ts`
- `types/shared.ts`
- `types/customer.ts`
- `types/gql/definitions/` (`base`, `supervisor`)
- `types/gql/gql-types/` (`base`, `supervisor`)

Rule:
- when a type becomes shared across multiple pages/components/services, promote it here,
- keep this folder focused on stable contracts rather than every local prop type in the project.
- backend-owned GraphQL SDL and generated type mirrors belong here, not under individual UI folders.

## 6) `src/app/helpers/` ? shared helpers, not UI ownership

Helpers are allowed here when they are:
- pure utilities,
- reusable across features,
- not better expressed as services, hooks, or components.

Examples:
- generic helpers,
- hash helpers,
- local ability/Joi helper area.

Rule:
- helpers should remain side-effect-light and reusable,
- UI composition should not drift into helpers,
- business workflows should not hide inside helper utilities if they actually belong to services/forms/adapters.

## 7) Relationship to `website/`

`cpanel/` should reuse the **organizational lessons** from `website/`, not its runtime shell.

### Shared concepts that should stay aligned

| Concept | `website/` | `cpanel/` translation |
|--------|-----------------|------------------------|
| UI foundation | `components/theme/Utils.tsx` | `app/ui/base/components/Utils.tsx` |
| Infra hooks/components | `components/base/` | `app/ui/base/` |
| Shared project UI | `components/` and `components/theme/` | `app/ui/components/` |
| Route-owned screens | `activities/` | `app/ui/pages/` |
| Shared data layer | adapters/hooks/injectors | adapters/hooks/injectors |
| Shared form layer | forms/hooks/injectors | forms/hooks/injectors |
| Local GraphQL tooling scope | owner-folder `graphql.config.yml` | subtree `graphql.config.yml` in `app/ui/pages/` and `app/ui/components/` when those trees host GraphQL-aware UI |
| Translation source | `resources/translations/*` | `resources/translations/*` |

### Cpanel conventions

- `MyPage` lifecycle and SSR boot via `web-core`.
- Supervisor GQL mirrors under `src/types/gql/**`.
The reusable part is the **mental model**:
- one design foundation,
- one infrastructure layer,
- one route layer,
- one data adapter layer,
- one form/requester layer,
- one translation source,
- one predictable ownership map.

## 8) Quick decision table

| If the code is... | Place it here |
|-------------------|---------------|
| SSR/client/server bootstrap | `src/client/` or `src/server/` |
| Platform route/store/axios/theme config | `src/resources/configs/` |
| Translation dictionary | `src/resources/translations/` |
| Global shell styles | `src/resources/emotion/styles/` |
| Router/auth/boot/global behavior service | `src/app/services/` |
| UI primitive or framework bridge tied to `web-core`/`sys-core` | `src/app/ui/base/` |
| Reusable project UI for cpanel screens | `src/app/ui/components/` |
| Shared page shell | `src/app/ui/layouts/` |
| Route-owned page entry | `src/app/ui/pages/` |
| Shared cross-feature type | `src/types/` |
| Pure helper utility | `src/app/helpers/` |

## 9) Final rule

When adding new control-panel functionality, always decide first:
1. is this platform config?
2. is this framework/base infrastructure?
3. is this shared project UI?
4. is this a route-owned page?
5. is this a service?
6. is this a shared type/helper?

If that ownership decision is not clear, stop and decide it before adding files.

## Related

- `docs/platforms/cpanel/ui-foundation.md`
- `docs/platforms/cpanel/data-flow-and-gql.md`
- `docs/platforms/cpanel/overview.md`
- `docs/invariants/cpanel.md`
