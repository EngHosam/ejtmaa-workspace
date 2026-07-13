# Website Auth Flow

Customer portal contract for auth flows (see `overview.md`).

## 1) Scope

Web-native **Customer** auth for `website/`: login, register, reset password, and Google social login/register.

## 1.1) Shipped state

Only **Login** ships. `Login.tsx` binds `useShallowForm({ initProps: { api: API.FORMS.R("auth") } })` (visitor), submits with `sub: "login"`, and on success calls `auth.login(myInstance, token)` → cookie + `window.location.reload()`. Fields: email + password.

Shipped visitor `auth` subs in `requesters.website.ts`: `registerCustomer`, `login`, `resetPassword`. `loginSocial` / `registerCustomerSocial` are **not** in the shipped requester map.

`Register`, `ResetPassword`, Google social, `CustomerHome`, `CUSTOMER_MAIN`, and the role-based redirect in §4A are **planned** (not shipped). `getMyHomeIdentify` currently returns `"Home"`; `publicRoutes = ["Login", "UiMockup", "Home"]`.

The remainder of this document describes the target auth contract.

Visitor auth requesters are consumed via `API.FORMS.R("auth")` (shipped visitor builder). Shipped subs in `website/src/types/requesters/requesters.website.ts`:

- `login`
- `registerCustomer`
- `resetPassword`

Planned subs (not yet in the shipped requester map):

- `loginSocial`
- `registerCustomerSocial`

Backend `AuthRequester.ts` gates auth subs for `["website"]` (`visitor` role).

## 2) Routes and pages

- Routes (`website/src/resources/configs/routes.ts`): `Login` -> `/login`, `Register` -> `/register`, `ResetPassword` -> `/reset-password` on `BASIC`.
- Role home: `CustomerHome` -> `/customer` on `CUSTOMER_MAIN` with `mustAuthedAs: ["CUSTOMER"]`.
- Pages: `Login.tsx`, `Register.tsx`, `ResetPassword.tsx` -> each is a `MyPage` with route-scoped `useShallowForm({ initProps: { api: API.FORMS.VISITOR.R("auth") } })`.
- Customer home: `pages/customer/Home.tsx` (`CustomerHome`).

## 3) UI components

- `AuthPageShell`, `AuthTextField`, `FormActionButton`, `FormTextField`, `SelectableCard`, `Checkbox`, `Logo`, `Loadable`.
- Primary actions use `SemanticColors.primary` (solid navy); Google button uses neutral tone.

## 4) Web-native behavior

- **Form injection** -- `FormTextField` stays `readOnly` until the shallow form exists (`!input.onChangeHandler`). Auth create forms do not wrap the page in `Loadable` during injection.
- **Email-only copy** -- all auth labels/subtitles use email only.
- **Internal navigation** -- `useNav().push` between auth pages; full reload only via `auth.login` / `auth.logout`.
- **Register payload** -- customer email registration sends `email` + `password`; social register deletes email/password and sends cached `social` token; terms `accepted` required.
- **Post-login** -- `auth.login(myInstance, token)` -> cookie + `window.location.reload()`.

## 4A) Role-based home redirect

- `publicRoutes`: `Login`, `Register`, `ResetPassword`, `UiMockup`, `Home`.
- `getMyHomeIdentify`: `CustomerHome` when `authedAs === "CUSTOMER"`, else `Home`.
- Authed on a public route -> redirect to `CustomerHome`.
- Unauthed on `/customer/*` -> redirect to `Home`.
- Depends on `/website/custom/start` returning auth under the `auth` key.

## 5) Social auth (Google)

- Google Identity Services One Tap via `SocialAuthHelpers.ts`.
- ID token forwarded to `loginSocial` / `registerCustomerSocial`.
- Retry caching in form values to avoid GIS re-prompt on validation errors.

## 6) Verification checklist

- Auth routes on `BASIC` with route-scoped shallow forms.
- Successful auth calls `auth.login` -> cookie + reload.
- Google social uses cached ID token on retry.
- `getMyHomeIdentify` returns `CustomerHome` for customers.
- Auth UI composes from `Utils` + semantic tokens.
- `yarn type-check` passes.

## 7) Related

- `docs/platforms/website/ssr-boot-and-startup.md`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/invariants/website.md` (W8, W9)
- `.cursor/rules/website-auth-flow.mdc`
