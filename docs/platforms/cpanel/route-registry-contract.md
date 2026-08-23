# CPanel Route Registry Contract

Supervisor control-panel contract (see `overview.md`).

Authoritative contract for `cpanel/src/resources/configs/routes.ts` and the mirrored `MPagesRoutes` interface in the same file.

Related runtime: `cpanel/src/app/services/router.ts` (`publicRoutes`, `applyRouterMiddleware`, `getMyHomeIdentify`).

## 1) Purpose

- Single routing authority for the SSR Supervisor cpanel (`@my-ssr/web-core` first-match router) on backend mount `/cpanel`.
- Namespace authed workspace URLs under `/supervisor/*`.
- Keep `routes` object keys and `MPagesRoutes` members grouped and ordered: **public → supervisor → base**.

## 1.1) Shipped state

| Identify | Path | Layout | Auth |
|---|---|---|---|
| `Login` | `/login` | `BASIC` | public (`publicRoutes`) |
| `Home` | `/` | `BASIC` | registered occupancy only; **not** in `publicRoutes` |
| `SupervisorHome` | `/supervisor` | `SUPERVISOR_MAIN` | `mustAuthedAs: ["SUPERVISOR"]` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` | middleware early-return |

`MPagesRoutes` mirrors the same identifies (`Error.error: number`). Layouts shipped: `BasicLayout` (`BASIC`), `SupervisorMainLayout` (`SUPERVISOR_MAIN`). There is no `MAIN`, `LANDING`, or `UiMockup` route.

`publicRoutes = ["Login"]`. `getMyHomeIdentify` always returns `"SupervisorHome"`.

## 2) Path helper (mandatory for workspace routes)

Declared once at the top of `routes.ts` (after imports, before `const routes`):

```ts
const supervisorRouter = (path: string) => `/supervisor${path}`;
```

| Helper | Use |
|--------|-----|
| `supervisorRouter("")` | Supervisor role home → `/supervisor` |
| `supervisorRouter("/…")` | Later supervisor feature paths |

**Rules:**

1. Every `mustAuthedAs: ["SUPERVISOR"]` route path MUST go through `supervisorRouter(...)`.
2. `Login`, `Home`, and `Error` keep absolute paths — do **not** wrap them in the helper.
3. Do **not** hand-write `/supervisor/...` literals on workspace routes when the helper applies.
4. Register fixed `/supervisor/...` segments before parametric `/supervisor/:id` on the same prefix.

In-app navigation uses `nav.push({ identify, … })`. Auth logo on `AuthPageShell` links to identify `SupervisorHome`.

## 3) Registry section blocks

Both `const routes` and `export interface MPagesRoutes` use **public → supervisor → base**, with `Error` last.

## 4) `Home` at `/` (occupancy, not a landing)

`Home` exists so `/` is a matched route (the Error pattern does not catch the empty path). `cpanel/src/app/ui/pages/Home.tsx` `Main()` returns `null`. It is a `MyPage` so `applyRouterMiddleware` still runs.

`Home` is **not** in `publicRoutes`:

- Unauthenticated `/` → redirect `Login`.
- Authenticated `/` stays on the empty `Home` page (no bounce to `SupervisorHome`). Authed supervisors reaching workspace home go through `Login` redirect or in-shell navigation to `SupervisorHome`.

Do not reuse the Login page on `/`. Do not treat `Home` as a public marketing landing (Website `Home` is `LANDING`; CPanel `Home` is occupancy only).

## 5) Runtime redirects

| Condition | Result |
|---|---|
| Identify `Error` | middleware returns; no auth bounce |
| Authed on `Login` | `SupervisorHome` (`replace`) |
| Unauthed on any identify except `Login` (includes `Home`, `SupervisorHome`) | `Login` (`replace`) |
| Authed without permission | `NOT_PERMIT` remote error |

SSR boot (`cpanel/src/app/services/index.ts`): `GET /custom/start` → `setRouterAccessPermission` → `auth.loadCurrentSupervisor` (GQL `me`, skipped when not authed as `SUPERVISOR`).

## 6) Traceability (this change set)

| Path | Role | Documented |
|---|---|---|
| `cpanel/src/resources/configs/routes.ts` | Registry + `supervisorRouter` + `MPagesRoutes` | this page |
| `cpanel/src/app/services/router.ts` | `publicRoutes`, home identify, middleware | §4–§5; `login-runtime-and-feedback.md` |
| `cpanel/src/app/ui/pages/Home.tsx` | Empty occupancy page | §4 |
| `cpanel/src/app/ui/pages/Login.tsx` | Login (unchanged page class this slice) | `login-runtime-and-feedback.md` |
| `cpanel/src/app/ui/pages/supervisor/SupervisorHome.tsx` | Authed home page | `flow-supervisor-shell.md` |
| `cpanel/src/app/ui/pages/Error.tsx` | Error CTA → `SupervisorHome` | `error-route-and-guard.md` |

## Related

- `docs/platforms/cpanel/flow-supervisor-shell.md`
- `docs/platforms/cpanel/overview.md`
- `.cursor/rules/cpanel-route-registry-governance.mdc`
