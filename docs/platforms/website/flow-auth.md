# Website Auth Flow

Customer portal contract for visitor authentication on `website/` (see `overview.md`).

This document describes the **current shipped state**. Planned extensions (Google social, customer role home redirect) are listed in §8 only.

## 1) Scope

Web-native **visitor** auth for `website/`: login, register, and reset password.

- Layout: `BASIC` (`BasicLayout`) for all three auth pages.
- Form API: `API.FORMS.R("auth")` (visitor requester builder).
- Backend gate: `AuthRequester.ts` subs `@sub(["website"], ["visitor"])`.
- Shipped visitor `auth` subs in `requesters.website.ts`: `registerCustomer`, `login`, `resetPassword`.

Not shipped: `loginSocial`, `registerCustomerSocial`, terms checkbox, Google GIS UI.

## 2) Routes and runtime access

| Identify | Path | Layout | Page |
|---|---|---|---|
| `Login` | `/login` | `BASIC` | `src/app/ui/pages/Login.tsx` |
| `Register` | `/register` | `BASIC` | `src/app/ui/pages/Register.tsx` |
| `ResetPassword` | `/reset-password` | `BASIC` | `src/app/ui/pages/ResetPassword.tsx` |

Router (`src/app/services/router.ts`):

- `publicRoutes = ["Login", "Register", "ResetPassword", "UiMockup", "Home"]`.
- `getMyHomeIdentify` returns `"CustomerHome"` when `authedAs === "CUSTOMER"`, else `"Home"`.
- Authed user on a public route → redirect to `getMyHomeIdentify` (`CustomerHome` for customer).
- Unauthed user on a non-public route → redirect to `Login`.

## 3) Page composition pattern

Each auth page is a `MyPage` with the same structural contract:

1. **`Main()`** — route-scoped `useShallowForm({ initProps: { api: API.FORMS.R("auth") } })`; exposes `formIdentify`, `d`, `status`; wraps inner card in `FormProvider`.
2. **Inner form card** — functional component receiving `canSubmit = form.exist` and `loading = form.status === "SENDING"`.
3. **`<Helmet><title>{pageTitle}</title></Helmet>`** — document title from `ui.pages.{login|register|resetPassword}.pageTitle`.
4. **`AuthPageShell`** — split brand panel + white form card (see §4).
5. **`<Box As={"form"}>`** — `onSubmit` calls `e.preventDefault()` then the page send handler.
6. **Submit** — `FormActionButton` primary, `disabled={!canSubmit}`, `loading={loading}`, `fullWidth`.
7. **No page-level `Loadable`** during form injection; spinner only on the submit button.

## 4) AuthPageShell layout contract

File: `src/app/ui/components/auth/AuthPageShell.tsx`.

### 4.1) Split layout

- **Brand panel** (left on desktop, above form on mobile): logo, eyebrow pill, `brandTitle`, `brandLead`, optional `trustBadges`.
- **Form card** (right on desktop, below brand on mobile): white `cardBackground`, `shellBorder`, padding `2rem 2.15rem`, **no shadow**.
- Breakpoint: `landingShellBreakpointPx` (1100) from `src/app/ui/layouts/landing-layout/shell.ts`.
- Grid: `cols={{ default: 2, 1100: 1 }}` when brand panel is shown; `ai_fs` on mobile, `alignItems: center` on desktop via `landingShellDesktopMedia`.

### 4.2) Outer shell and decorative corners

The outer wrapper matches the landing `Hero` section contract — **not** `minH` + vertical centering:

- `relative`, `clp`, `p={"3rem 0 4rem"}`.
- Desktop padding override: `paddingTop: 5rem`, `paddingBottom: 6rem` via `landingShellDesktopMedia`.
- Decorative corner brackets: navy top-right (`primaryActionBackground`), orange bottom-left (`accentActionBackground`), same motion as `Hero.tsx`.

This keeps corner brackets anchored to the content section on mobile.

### 4.3) Logo navigation

Default brand logo (`Logo preset="auth"`) is wrapped in `Box As={Link}` with `href: { identify: "Home" }`. `title` and `aria-label` use `t.app("title")`. Custom `brandSlot` overrides are caller-owned.

### 4.4) Shell props

| Prop | Role |
|---|---|
| `brandSlot` | Optional logo/brand override in brand panel |
| `eyebrow`, `brandTitle`, `brandLead`, `trustBadges` | Brand panel copy |
| `title`, `description` | Form card heading |
| `actionArea`, `footerSlot` | Optional slots below form body |
| `children` | Form fields and actions |

Theme and language switches live in `BasicLayout`, not in `AuthPageShell`.

## 5) Auth UI components

| Component | Path | Role |
|---|---|---|
| `AuthPageShell` | `components/auth/AuthPageShell.tsx` | Split layout shell |
| `AuthTextField` | `components/auth/AuthTextField.tsx` | Thin wrapper over `FormTextField` (`text` \| `email` \| `password` \| `tel`) |
| `AuthNavLink` | `components/auth/AuthNavLink.tsx` | Typed `Link` to `Login` \| `Register` \| `ResetPassword`; used for forgot-password |
| `AuthSecondaryNavButton` | `components/auth/AuthSecondaryNavButton.tsx` | Hint text + `FormActionButton tone="secondary"`; navigates via `useNav().push({ identify })` |
| `FormTextField` | `components/form/FormTextField.tsx` | Shared field primitive |
| `FormActionButton` | `components/form/FormActionButton.tsx` | Primary / secondary / neutral submit buttons |
| `FormInputWrapper` | `components/form/FormInputWrapper.tsx` | Label, optional `actionArea`, errors, input chrome |
| `Logo` | `components/Logo.tsx` | `auth` preset `9.5rem` height |

### 5.1) Form field contract

- `FormTextField` sets `readOnly: !input.onChangeHandler` until shallow form injection completes.
- Auth pages do not pass `placeholder` (default `""`).
- Password field on Login passes `actionArea={<AuthNavLink identify="ResetPassword" compact />}` for forgot-password link aligned in the field header row.
- Email/password inputs use `ltr` on the native `<input>` (existing `FormTextField` behavior).

### 5.2) Button tones

| Tone | Surface | Use on auth pages |
|---|---|---|
| `primary` | `primaryActionBackground` / `primaryActionText` | Submit (login, register, reset) |
| `secondary` | `secondaryActionBackground` / `secondaryActionText` | Cross-auth CTA (create account, sign in, back to login) |
| `neutral` | `cardBackground` + `inputBorder` | Not used on shipped auth pages |

Secondary tone has no shadow on hover (unlike primary).

## 6) Per-page behavior

### 6.1) Login

- **Sub:** `login`
- **Fields:** `email`, `password`
- **Success:** `res.data?.token` → `auth.login(myInstance, token)` → cookie + `window.location.reload()`
- **Cross-nav:** `AuthNavLink` → `ResetPassword`; `AuthSecondaryNavButton` → `Register`
- **i18n:** `ui.pages.login` (+ `fields.email`, `fields.password`)

### 6.2) Register

- **Sub:** `registerCustomer`
- **Fields:** `name`, `email`, `mobile`, `password` — matches backend `RegisterCustomerProps` (`AuthRequester.ts`: name min 4, email, mobile, password min 8). No `accepted` terms field (not in backend website sub).
- **Success:** `nav.push({ identify: "Login" })` — backend returns no token; may send verification email when `EMAIL_VERIFY_ENABLED`.
- **Cross-nav:** `AuthSecondaryNavButton` → `Login`
- **i18n:** `ui.pages.register` (+ field namespaces)

### 6.3) ResetPassword

- **Sub:** `resetPassword`
- **Fields:** `email`
- **Success:** `nav.push({ identify: "Login" })`
- **Cross-nav:** `AuthSecondaryNavButton` → `Login` (no hint prop)
- **i18n:** `ui.pages.resetPassword` (+ `fields.email`, `description`)

## 7) Navigation rules

| Target | Mechanism |
|---|---|
| Auth page ↔ auth page (secondary CTA) | `useNav().push({ identify })` via `AuthSecondaryNavButton` |
| Forgot password (inline link) | `AuthNavLink` → `Link` with `href: { identify }` |
| Logo on auth pages | `Link` with `href: { identify: "Home" }` |
| Post-login session | `auth.login` → full reload only |
| Post-logout | `auth.logout` → full reload only |

No raw path strings (`/login`, `/register`) in auth UI. Typed `identify` values must exist in `MPagesRoutes`.

## 8) Landing integration

- `LandingHeader` register CTA: `push({ identify: "Register" })`.
- `LandingLayout` mobile drawer register action: `push({ identify: "Register" })`.
- Login CTA on landing: `push({ identify: "Login" })`.

## 9) Copy and i18n

Namespaces (both `ar.ts` and `en.ts`):

- `ui.pages.login` — `pageTitle`, `eyebrow`, `brandTitle`, `lead`, `title`, `submit`, `noAccount`, `createAccount`, `forgotPassword`, `fields.*`
- `ui.pages.register` — `pageTitle`, `eyebrow`, `brandTitle`, `lead`, `title`, `submit`, `hasAccount`, `signIn`, `fields.*`
- `ui.pages.resetPassword` — `pageTitle`, `eyebrow`, `brandTitle`, `lead`, `title`, `description`, `submit`, `backToLogin`, `fields.*`

No placeholder keys on auth field namespaces.

## 10) Planned (not shipped)

- Google social auth (`SocialAuthHelpers.ts`, `loginSocial`, `registerCustomerSocial`).
- `CustomerHome` → `/customer` on `CUSTOMER_MAIN` (shipped); `getMyHomeIdentify` returns `CustomerHome` for `authedAs === "CUSTOMER"`.
- Terms `accepted` checkbox (only if backend website `registerCustomer` adds the field).
- `SelectableCard` provider picker on Register.

## 11) Verification

- Auth routes on `BASIC` with route-scoped shallow forms.
- `publicRoutes` includes `Login`, `Register`, `ResetPassword`, `Home`.
- Login success calls `auth.login` → cookie + reload.
- Register/ResetPassword success navigates to `Login` without reload.
- `FormTextField` `readOnly` during injection.
- `yarn type-check` passes in `website/`.

## 12) Related

- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/website/brand-identity-alignment.md`
- `docs/platforms/website/component-structure.md`
- `docs/invariants/website.md` (W8, W9)
- `.cursor/rules/website-auth-flow.mdc`
