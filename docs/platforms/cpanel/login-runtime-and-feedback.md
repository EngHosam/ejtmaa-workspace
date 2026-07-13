# CPanel Login Runtime and Feedback Surfaces

Target supervisor cpanel contract (see `overview.md`). Describes login flow and shared auth/feedback surfaces for the supervisor cpanel contract.

## Purpose

This page documents the cpanel login flow, router-middleware timing, and shared auth/feedback UI surfaces used by the control panel.

## Outcomes

Contract outcomes when the checkout implements supervisor login:

- `Login` renders a supervisor login form on `BASIC`.
- The login form posts to visitor-scoped cpanel requester `auth/supervisorLogin`.
- Successful login writes the returned token to the cpanel cookie store and reloads the document so the normal boot path owns startup hydration.
- `/cpanel/custom/start` returns `data.auth`, matching the cpanel auth reducer contract.
- Router middleware applies from `MyPage` lifecycle hooks (`WillServerRender`, `WillClientHydrate`, `WillClientNavigate`).
- Shared auth UI lives under `cpanel/src/app/ui/components/auth/`.
- `AuthPageShell` owns the login card width boundary through its shared `Container` contract and reduces card padding on `xs` screens.
- `Login` renders fields/actions under `AuthPageShell` with submission from the shared action button.
- Shared feedback UI covers `Loadable`, `GlobalLoadable`, and `Toast`.
- `UiMockup` includes visual-review sections for loadable and toast surfaces plus shell/table review blocks.

## Runtime Ownership

### Startup hydration remains owned by `services/index.ts`

`cpanel/src/app/services/index.ts` remains the single startup owner for the cpanel auth/global bootstrap:

- server boot calls `GET /cpanel/custom/start`,
- server boot dispatches the returned startup payload through `global.setServerStartData(...)`,
- server boot computes route access permission after startup data has been written,
- client boot only finishes CSR status, socket preparation, and pre-message replay.

This means login does **not** hydrate auth state manually. the flow intentionally re-enters the normal startup path by reloading the page after the token is stored.

### Router middleware timing fix

The durable runtime contract is router middleware placement from `MyPage` lifecycle hooks.

Contract truth:

- `cpanel/src/resources/configs/web-core.ts` registers router middleware through `MyPage`, not `router.beforePageLoading`.
- `cpanel/src/app/ui/base/core/MyPage.tsx` applies `router.applyRouterMiddleware(...)` from:
  - `WillServerRender`
  - `WillClientHydrate`
  - `WillClientNavigate`

Reason for this placement:

- the prior `beforePageLoading` hook ran too early in the SSR prepare path,
- which allowed route auth/redirect checks to execute before `services.boot.server(...)` had loaded `/custom/start`,
- which meant the middleware could evaluate auth against incomplete startup state.

The placement makes route gating follow the page lifecycle, so the middleware sees startup state after the server boot sequence has populated it.

## Login Flow

### Route page

`cpanel/src/app/ui/pages/Login.tsx` owns the route-level login flow:

- uses `useShallowForm(...)` so the page gets a deterministic form boundary,
- initializes the form with `API.FORMS.R("auth")`,
- submits sub `supervisorLogin`,
- blocks duplicate submits when the form status is `SENDING`,
- passes loading state into the submit button,
- renders the input/action column directly without an extra wrapper node,
- triggers submit from `FormActionButton` `onClick` with `type="button"` instead of a page-local native submit wrapper,
- delegates successful token handling to `auth.login(...)`.

### Shared auth UI

The route page stays orchestration-oriented by composing shared project UI:

- `AuthPageShell.tsx` -> auth card shell with logo/title/action/footer slots plus the shared login-width boundary (`w="100%"`, `maxW="34rem"`, `minW={0}`) and compact `xs` padding
- `AuthTextField.tsx` -> form-bound input with autofill-safe styling and field errors
- `FormActionButton.tsx` -> shared gradient action button with inline loading state
- `FormInputWrapper.tsx` -> shared project field frame for title/subtitle/error/action areas

### Responsive width ownership

Responsive sizing lives in shared UI rather than in `Login.tsx`:

- `AuthPageShell.tsx` uses the shared `Container` primitive for auth-card width,
- the shell sets explicit `w`, `maxW`, and `minW` values on that shared `Container`,
- the inner auth card reduces padding on `xs` screens through `withMaxScreenWidth(...)`,
- `Login.tsx` stays orchestration-only by rendering the field/action column directly and leaving width ownership to the shared auth shell.

Login copy in `ar.ts` is minimal: page title, submit label, and field labels/placeholders only.

### Successful login behavior

`cpanel/src/app/services/auth.ts` does the following on success:

1. stores the returned token in the cookie manager for one year,
2. triggers `window.location.reload()`.

This is intentional in the implementation because startup hydration is owned by the normal boot path, not by a second manual client-side auth-hydration path.

## Backend Contract

### Requester

The login page consumes the already-existing backend requester:

- `POST /cpanel/forms/requester/auth/supervisorLogin`

The frontend requester typing and endpoint helpers expose this route:

- `cpanel/src/types/requesters/requesters.cpanel.ts`
- `cpanel/src/resources/configs/axios/api.ts`

Cpanel requester split:

- visitor:
  - `auth.supervisorLogin`
- supervisor:
  - `customer.read|update`
  - `platform_settings.read|update`
  - `supervisor.read|update`
  - `customer.read|update`
  - `website_settings.read|update`

### Startup response

`backend/src/app/http/controllers/cpanel/custom/StartController.ts` returns:

- `data.global`
- `data.auth`

not `data.authentication`.

This matches the cpanel startup dispatch path:

- `global.setServerStartData(...)` spreads startup payload directly into the global start action
- the cpanel auth reducer reads `action.auth`

See `docs/platforms/backend/contracts/http-and-requesters.md` for the backend HTTP contract.

## Shared Feedback Surfaces

### Loadable

`cpanel/src/app/ui/components/Loadable.tsx` is the shared feedback loading surface.

Behavior:

- uses `lottie-react`,
- chooses `dark-loading.json` for light backgrounds,
- chooses `light-loading.json` for dark backgrounds,
- supports `inline` and `overlay` variants,
- supports caller-controlled size and optional background token.

### GlobalLoadable

`cpanel/src/app/ui/components/GlobalLoadable.tsx` reads `state.global.busy` through `useSector(...)` and renders `Loadable` as a full-screen overlay with `semanticColor.softBackdrop`.

`cpanel/src/app/services/global.ts` exposes `switchGlobalLoading(...)`, which dispatches `Actions.GLOBAL.BUSY_SWITCH`.

### Toast

`cpanel/src/app/ui/components/Toast.tsx` and `cpanel/src/app/services/toast.tsx` form the shared toast surface.

Behavior:

- service accepts the existing `Message` shape,
- service renders `Toast` through `react-toastify`,
- toast position follows current RTL state,
- success uses the bundled `success.json` Lottie asset instead of a static icon,
- info/warning/error use semantic icon chips,
- toast sizing was deliberately increased during implementation to improve presence and icon readability,
- service clears the default `react-toastify` card styling so the project-owned toast surface controls presentation.

## Visual Review Route

`cpanel/src/app/ui/pages/UiMockup.tsx` reviews:

- `BasicLayout`
- `MainLayout`
- `ThemeModeSwitch`
- `Header`
- `Drawer`
- `Footer`
- `Loadable`
- `Toast`
- `DataTable`

Loadable/toast review behavior:

- inline loadable preview on light background
- inline loadable preview on dark background
- live trigger for the global overlay loadable
- live trigger buttons for real toasts through `toastService.showToast(...)`
- inline toast rendering for all message statuses

## Theme and Type System Adjustments

### Theme typing and gradients

`cpanel/src/resources/configs/theme.ts` (when present in cpanel checkout) provides shared gradient and element-reset surfaces for auth/button/theme-switch work:

- semantic gradient key `segmentedSelected`
- `getSemanticGradient(...)`
- `getGradientBackground(...)`
- `ElementStyles.buttonReset`

It keeps `ThemeMapPath` derived from `FullNestedPaths` while typing `DarkSchema` through `FixTypes<typeof LightSchema>`.

### Generic DataTable row typing

`cpanel/src/app/ui/components/DataTable.tsx` uses generic row typing:

- `DataTableColumn<TRow>`
- `DataTableProps<TRow>`

This allows route-local mock rows such as `MockRequestRow` in `UiMockup` to provide strongly typed custom cell renderers without forcing every table renderer back to the base row shape.

## Generated and Asset Paths

The following inventoried paths are verification assets and are not narrated line-by-line:

- `cpanel/src/resources/animations/dark-loading.json`
- `cpanel/src/resources/animations/light-loading.json`
- `cpanel/src/resources/animations/success.json`
- `cpanel/lib/tsconfig.tsbuildinfo`

Rationale:

- the animation files are static Lottie asset payloads consumed by shared UI components,
- `tsconfig.tsbuildinfo` is a generated incremental TypeScript artifact and is not source-of-truth implementation code.

## Contract inventory

The table below inventories every tracked path for the login surface contract.

| Path | Contract role | Where documented |
|------|---------------|------------------|
| `cpanel/src/app/ui/components/auth/AuthPageShell.tsx` | Shared `Container` owns login card width; reduced auth-card padding on `xs` screens. | `# Login Flow` / `### Shared auth UI`; `# Login Flow` / `### Responsive width ownership` |
| `cpanel/src/app/ui/pages/Login.tsx` | Route-owned login fields with submit via `FormActionButton` `onClick`. | `# Login Flow` / `### Route page`; `# Login Flow` / `### Responsive width ownership` |
| `cpanel/src/resources/configs/web-core.ts` | Web-core config version `0.2`. | `# Runtime Ownership` / `### Router middleware timing fix` |
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated incremental TypeScript verification artifact. | `# Generated and Asset Paths` |

## Contract inventory (full surface)

The table below accounts for the broader cumulative surface documented by this page, including generated/assets/docs/rules paths that require explicit accounting.

| Path | Role in the documented surface | Where documented |
|------|------------------------------|------------------|
| `cpanel/src/app/services/auth.ts` | Token storage and post-login reload entrypoint | `# Login Flow` / `### Successful login behavior` |
| `cpanel/src/app/services/global.ts` | Startup dispatch helpers plus global busy switch | `# Runtime Ownership` / `### Startup hydration remains owned by services/index.ts`; `# Shared Feedback Surfaces` / `### GlobalLoadable` |
| `cpanel/src/app/services/index.ts` | Canonical startup owner for `/custom/start` and access-permission setup | `# Runtime Ownership` / `### Startup hydration remains owned by services/index.ts` |
| `cpanel/src/app/services/router.ts` | Route auth/redirect middleware implementation | `# Runtime Ownership` / `### Router middleware timing fix` |
| `cpanel/src/app/services/toast.tsx` | Shared toast service using `react-toastify` and project-owned toast UI | `# Shared Feedback Surfaces` / `### Toast` |
| `cpanel/src/app/ui/base/core/MyPage.tsx` | Page lifecycle owner for `applyRouterMiddleware(...)` | `# Runtime Ownership` / `### Router middleware timing fix` |
| `cpanel/src/app/ui/components/DataTable.tsx` | Generic shared table surface used by the review route | `# Theme and Type System Adjustments` / `### Generic DataTable row typing`; `# Visual Review Route` |
| `cpanel/src/app/ui/components/GlobalLoadable.tsx` | Store-backed global loading overlay | `# Shared Feedback Surfaces` / `### GlobalLoadable` |
| `cpanel/src/app/ui/components/Loadable.tsx` | Shared Lottie loading surface | `# Shared Feedback Surfaces` / `### Loadable` |
| `cpanel/src/app/ui/components/ThemeModeSwitch.tsx` | Shared theme-mode switch using gradient-selected state | `# Theme and Type System Adjustments` / `### Theme typing and gradients`; `# Visual Review Route` |
| `cpanel/src/app/ui/components/Toast.tsx` | Shared cpanel toast UI with success animation | `# Shared Feedback Surfaces` / `### Toast` |
| `cpanel/src/app/ui/components/form/FormInputWrapper.tsx` | Shared auth/project field wrapper with errors/title/action area | `# Login Flow` / `### Shared auth UI` |
| `cpanel/src/app/ui/components/auth/AuthPageShell.tsx` | Shared auth shell card | `# Login Flow` / `### Shared auth UI` |
| `cpanel/src/app/ui/components/auth/AuthTextField.tsx` | Shared auth input field component | `# Login Flow` / `### Shared auth UI` |
| `cpanel/src/app/ui/components/form/FormActionButton.tsx` | Shared action button used by the login route and other requester-backed cpanel forms | `# Login Flow` / `### Shared auth UI` |
| `cpanel/src/app/ui/layouts/BasicLayout.tsx` | Lightweight layout embeds `ThemeModeSwitch` for auth/error-like routes | `# Visual Review Route` |
| `cpanel/src/app/ui/pages/Login.tsx` | Route-owned supervisor login orchestration | `# Login Flow` / `### Route page` |
| `cpanel/src/app/ui/pages/UiMockup.tsx` | Visual-review route for shell, loadable, toast, and table states | `# Visual Review Route` |
| `cpanel/src/resources/configs/axios/api.ts` | Truthful frontend cpanel requester/custom endpoint map | `# Backend Contract` / `### Requester` |
| `cpanel/src/resources/configs/theme.ts` | Shared gradients, reset styles, typed theme map, and dimension authority (when present in cpanel checkout) | `# Theme and Type System Adjustments` / `### Theme typing and gradients` |
| `cpanel/src/resources/configs/web-core.ts` | Central runtime config authority; middleware placement and version `0.2` | `# Runtime Ownership` / `### Router middleware timing fix`; `# Contract inventory` |
| `cpanel/src/resources/translations/ar.ts` | Centralized login/mockup/toast/loadable Arabic copy | `# Login Flow` / `### Shared auth UI`; `# Visual Review Route` |
| `cpanel/src/types/requesters/requesters.cpanel.ts` | Truthful cpanel requester type map for visitor/supervisor scopes | `# Backend Contract` / `### Requester` |
| `cpanel/src/resources/animations/dark-loading.json` | Lottie asset for loaders on light backgrounds | `# Generated and Asset Paths` |
| `cpanel/src/resources/animations/light-loading.json` | Lottie asset for loaders on dark backgrounds | `# Generated and Asset Paths` |
| `cpanel/src/resources/animations/success.json` | Lottie asset for success toast state | `# Generated and Asset Paths` |
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated incremental TypeScript artifact; not source-of-truth logic | `# Generated and Asset Paths` |
| `backend/src/app/http/controllers/cpanel/custom/StartController.ts` | Backend `/cpanel/custom/start` response returns `auth` | `# Backend Contract` / `### Startup response` |
| `docs/platforms/backend/contracts/http-and-requesters.md` | Backend HTTP contract for cpanel start response shape | `# Backend Contract` / `### Startup response` |
| `docs/platforms/cpanel/login-runtime-and-feedback.md` | Login/runtime contract page | this page |
| `docs/platforms/cpanel/README.md` | Cpanel documentation index | `docs/platforms/cpanel/README.md` |
| `docs/platforms/cpanel/overview.md` | Platform overview for login/runtime and shared-feedback surfaces | `docs/platforms/cpanel/overview.md` |
| `docs/platforms/cpanel/repository-inventory.md` | Repository inventory for login/runtime file roles | `docs/platforms/cpanel/repository-inventory.md` |
| `docs/platforms/cpanel/component-structure.md` | Ownership guidance for middleware timing and shared auth/feedback UI | `docs/platforms/cpanel/component-structure.md` |
| `docs/platforms/cpanel/ui-foundation.md` | UI foundation for auth/loadable/toast surfaces | `docs/platforms/cpanel/ui-foundation.md` |
| `docs/platforms/cpanel/data-flow-and-gql.md` | Data-flow contract for login form/requester usage and startup alignment | `docs/platforms/cpanel/data-flow-and-gql.md` |
| `.cursor/rules/cpanel-platform-governance.mdc` | Durable governance for router-middleware timing and `AuthPageShell` width ownership | `.cursor/rules/cpanel-platform-governance.mdc` |
