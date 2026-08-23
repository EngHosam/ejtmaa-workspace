# CPanel Login Runtime and Feedback

Supervisor login on `cpanel/` (see `overview.md`). This page describes the **current shipped** login path. It is the Website auth page DNA with visitor `sub` remapped to `supervisorLogin` and customer register/reset links removed.

## 1) Scope

- Layout: `BASIC` (`BasicLayout`).
- Form API: `API.FORMS.R("auth")` (visitor requester builder on `/cpanel`).
- Backend: visitor `auth` sub `supervisorLogin` on the cpanel platform.
- Success: `auth.login(myInstance, token)` writes the cookie and reloads so `services.boot.server` owns hydration.

Not shipped: register, reset-password, social login, extra auth CTAs.

## 2) Route and access

| Identify | Path | Layout | Page |
|---|---|---|---|
| `Login` | `/login` | `BASIC` | `src/app/ui/pages/Login.tsx` |
| `Home` | `/` | `BASIC` | `src/app/ui/pages/Home.tsx` (renders `null`) |

Router (`src/app/services/router.ts`):

- `publicRoutes = ["Login"]`.
- `getMyHomeIdentify` returns `"SupervisorHome"` (`/supervisor`).
- Authed supervisor on `Login` → redirect to `SupervisorHome`.
- Unauthed visitor on any other identify (including `Home`) → redirect to `Login`.
- `Error` exits middleware immediately (`currentIsErrorPage`).

## 3) Page composition

`Login` is a `MyPage` using the Website auth structural contract:

1. **`Main()`** — `useShallowForm({ initProps: { api: API.FORMS.R("auth") } })`; wraps the inner card in `FormProvider`.
2. **Inner card** — `canSubmit = form.exist`, `loading = form.status === "SENDING"`.
3. **`<Helmet><title>`** — `ui.pages.login.pageTitle`.
4. **`AuthPageShell`** — split brand panel + white form card (Website DNA; not a generic max-width `Container` admin card).
5. **`<Box As={"form"}>`** — `preventDefault` then `d.send`.
6. **Submit** — `FormActionButton` with `disabled={!canSubmit}` and `loading={loading}`.
7. **Send** — `sub: "supervisorLogin"`; `funnels.afterSentSuccess` reads `res.data?.token` and calls `auth.login`.

No register / forgot-password controls.

## 4) Boot and middleware timing

`src/app/services/index.ts` owns startup:

- server: `GET` `API.CUSTOM.START` (`/custom/start` on mount `/cpanel`), then `global.setServerStartData`, then `router.setRouterAccessPermission`.
- client: `socket.prepareSocket`, `global.setClientIsStarted`, `preMessages.showPreMessagesOnBrowser`.

`src/resources/configs/web-core.ts` does not register router middleware on `router.beforePageLoading`.

`src/app/ui/base/core/MyPage.tsx` applies `router.applyRouterMiddleware(...)` from `WillServerRender`, `WillClientHydrate`, and `WillClientNavigate`.

## 5) Related

- `docs/platforms/website/flow-auth.md` — sibling auth DNA
- `docs/platforms/cpanel/overview.md`
- `docs/platforms/cpanel/error-route-and-guard.md`
- `.cursor/rules/cpanel-platform-governance.mdc`
