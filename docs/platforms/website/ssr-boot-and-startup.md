# Website SSR Boot and Startup

## Scope

Startup contract for `website/`: SSR boot, `/website/custom/start` server prepare, `MyPage` lifecycle, and cookie-token auth reload.

## 2) SSR boot pipeline

Boot sequence for the customer portal:

`website/src/app/services/index.ts` defines separate server and client boot phases:

- **Server phase** (`boot.server`):
  - early-exits on the error page,
  - calls `API.CUSTOM.START` (`/website/custom/start` via axios base) with `throwMode` + `throwWithErrorsFunnel` + `autoShowMainMessages`,
  - dispatches the startup payload through `global.setServerStartData(myInstance, res.data)`,
  - computes route access permissions via `router.setRouterAccessPermission(myInstance)` **after** startup data is written.
- **Client phase** (`boot.client`):
  - early-exits on the error page (marks client started and returns),
  - prepares socket via `socket.prepareSocket(myInstance)`,
  - marks the client started via `global.setClientIsStarted(myInstance)`,
  - shows pre-messages (1500ms delay before display).

Entry points:
- `src/client/index.ts`, `src/client/worker.ts`
- `src/server/index.ts`, `src/server/worker.ts`
- `src/resources/configs/web-core.ts` (config version `0.1`)
- `src/app/ui/base/core/MyApp.tsx`, `src/app/ui/base/core/MyHtml.tsx`, `src/app/ui/base/core/MyPage.tsx`

## 3) Router middleware timing (governance)

Router middleware MUST be applied from the `MyPage` lifecycle (`WillServerRender` / `WillClientHydrate` / `WillClientNavigate`), NOT from `web-core.router.beforePageLoading`. The earlier hook can run before `services.boot.server(...)` finishes `/website/custom/start` hydration during SSR prepare flow, so auth/route-access checks must run after startup data is loaded. See `docs/platforms/cpanel/login-runtime-and-feedback.md` for the same middleware placement rule on `/cpanel`.

## 4) Auth hydration

`website/src/app/services/auth.ts` uses:
- `AuthedAs = "CUSTOMER"`,
- `login(myInstance, token)` sets the `token` cookie (1-year expiry) then calls `window.location.reload()`,
- `logout` clears the cookie and redirects to `Login`,
- `getToken` reads from the cookie manager,
- `canDoAction` resolves `customer.permissions` by `authedAs`.

Auth uses a **cookie token + page reload** so the normal SSR boot path owns hydration. The reload re-runs the full server prepare + `/website/custom/start` flow.

Start payload: `/website/custom/start` returns the auth payload under the **`auth`** key, and the website auth reducer spreads it as `...action.auth`. See `docs/platforms/backend/contracts/client-portal-http-website.md`.

Post-login landing: after `auth.login` reloads, the router middleware in `website/src/app/services/router.ts` redirects to `getMyHomeIdentify(myInstance)` — `CustomerHome` for a customer, `Home` for a visitor. See `flow-auth.md` §4A.

## 6) Verification checklist

- Server phase calls `/website/custom/start` before route access is computed.
- Route middleware runs from `MyPage` lifecycle, not `web-core.router.beforePageLoading`.
- `/website/custom/start` returns the auth payload under the `auth` key so `isAuthed` / `authedAs` populate.
- `login` sets the cookie token and reloads; post-login redirect uses `getMyHomeIdentify`.
- `logout` clears the cookie and redirects to `Login`.
- Error pages early-exit boot and render the `Error` route.
- `yarn type-check` passes.

## 7) Related

- `docs/platforms/website/overview.md`
- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/flow-auth.md`
- `docs/platforms/cpanel/login-runtime-and-feedback.md` (shared SSR pattern)
- `docs/invariants/website.md` (W2, W16)
