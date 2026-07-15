# Website Shared UI + Shell

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native shared UI primitives and shell for `website/`: document shell, actor-specific headers, side drawers, authed customer bottom bar (`CustomerBottomBar`), and shared presentational primitives -- composed from `Utils` and semantic theme tokens.

## 1.1) Shipped shell

Layouts: `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`) — no `CUSTOMER_MAIN` yet.

| Context | Components |
|---|---|
| Auth + error (`BASIC`) | `ThemeModeSwitch`, `LanguageSwitch` in `BasicLayout` top row; auth pages use `AuthPageShell` |
| Landing (`LANDING`) | `LandingHeader`, `LandingFooter`, `LandingMobileDrawer`, `home/*` sections |
| Main workspace (`MAIN`) | `Header`, `Drawer`, `Footer` via `MainLayout` |

Generic shell components: `Header`, `Drawer`, `Footer`, `Logo`, `ThemeModeSwitch`, `LanguageSwitch` (under `src/app/ui/components/`).

Main drawer (`src/app/ui/layouts/main-layout/drawer.ts`): one `business` section with `dashboard` (→ `Home`) + `logout` only.

`AuthedAs = "CUSTOMER"`; auth reducer exposes `auth.customer.permissions`.

`CustomerHeader`, `CustomerDrawer`, and `CustomerBottomBar` are planned (see `flow-customer-shell.md`).

## 2) Document shell

- Document shell owned by `MyHtml` + `MyApp`/`MyPage` lifecycle.
- `MyHtml` owns `<html>` / `<head>` / `<body>`, theme mode, `lang` + `dir`, fonts, and global CSS.
- Safe-area insets use CSS `env(safe-area-inset-*)` or theme padding tokens.
- Authed customer shell uses `Dims.bottomBarHeight` for bottom clearance. See `flow-customer-shell.md`.

## 3) Actor-specific headers — planned

Shipped header: generic `Header` (`src/app/ui/components/Header.tsx`) with `main`/`compact` variants. The actor-specific headers below are planned.

| Actor | Component | Contract |
|---|---|---|
| Visitor | `LandingHeader` | Visitor landing pages; pairs with `LandingSubHeader` where needed |
| Customer | `CustomerHeader` | `CUSTOMER_MAIN` workspace; main vs subpage variants documented in `flow-customer-shell.md` |

Shared header behavior (both components):

- Back behavior: `onBack` if provided, else `useNav().back()` / browser history back.
- Badge behavior: numeric badge (`1..99`, capped `99`), dot badge (`true`), none (`0`/missing).
- LTR-first authoring; runtime `dir` handles mirroring.
- Fixed dimensions from `theme.ts`/`utils.ts` tokens. `zIndex.header` above body content.
- Brand slot: the logo (`Logo preset="header"`) renders bare — no border/background/padding pill around it. The `main` variant places `resolvedBrandSlot` directly in the trailing row. See `brand-identity-alignment.md` § Logo and `.cursor/rules/website-logo-no-frame.mdc`.
- Trailing control cluster: `ThemeModeSwitch` + `LanguageSwitch` render in the trailing row for the `main` non-compact variant (gated by `showThemeSwitch`). Header icon buttons (`HeaderIconButton`: menu / back / `trailingAction` e.g. notifications) use `crn={semanticDims.card.radius}` with `semanticColor.inputBackground` + `inputBorder` + `iconPrimary` — same corner and surface tokens as the two toggles.

## 4) Side drawer — planned

Shipped drawer: generic `Drawer` + `main-layout/drawer.ts` (dashboard + logout only). The role-specific drawers below are planned.

- Visitor drawer: `components/LandingDrawer.tsx` (mobile, portaled). See `component-structure.md` and visitor shell pages.
- Authed customer drawer: `CustomerDrawer.tsx` -- portaled side drawer with route-gated nav items. See `flow-customer-shell.md` section 5.
- Role resolution from the `auth` Redux slice `authedAs` (`CUSTOMER`).
- Profile payload from `useMe()`: `name`, `avatar_url`.
- UX order: hero identity zone (brand, title/subtitle, theme + language toggles) -> navigation blocks -> utility footer (utility items / sign-out). Footer stays outside scroll body.
- Motion via `framer-motion`. Drawer actions are close-first.
- Brand slot in the hero identity zone: `Logo preset="drawer"` rendered bare inside `<Box as_c>` (alignment only, no frame). The drawer hero column is centered horizontally and vertically (`ai_c` + `ta_c` + `jc_c`): brand slot, title, subtitle, and the theme + language toggles all align to the center. The two toggles sit in one centered `Row` (`ThemeModeSwitch` + `LanguageSwitch`), gated by `showThemeSwitch`. See `brand-identity-alignment.md` § Logo.

## 5) Shared primitives

- **Button**: `Utils`-composed; `actionPrimary` solid navy (`semanticColor.primaryActionBackground`) for primary CTAs; RTL-safe arrow mirroring via CSS `dir`; loading state via `Loadable` overlay.
- **SegmentedPills**: tab/scope semantics; a11y label per option.
- **Loadable**: spinner/progress surface for initial loading, submit-busy, and overlay loaders.
- **Toast**: success/error notifications; translation-backed copy.
- **Empty / Guest / LaneFailed / Able**: shared `Utils`-built primitives with stacking rules documented in `ui-foundation.md`.
- **ThemeModeSwitch** / **LanguageSwitch**: paired single-toggle buttons. `ThemeModeSwitch` shows the **target** mode icon (`FiMoon` when light, `FiSun` when dark); `LanguageSwitch` shows the **target** language letters (`EN` when current is `ar`, `ع` when current is `en`) and switches via `changeLocale` (cookie + full reload). Both use `semanticColor.inputBackground` + `inputBorder`, corner `semanticDims.card.radius` (not pills), and live in the drawer hero identity zone, the header trailing cluster, and `BasicLayout`'s top-right row. See `brand-identity-alignment.md` § Canonical consumer pairings and `ui-foundation.md` § Localization & RTL.

## 6) Navigation stabilization

- Web navigation is URL-router-based (`useNav` + browser history) inside `MyPage` lifecycle.
- Tab-root customer pages: header menu opens `CustomerDrawer`. Detail/subpage routes keep back-only behavior.
- Drawer actions are close-first.

## 7) Related

- `docs/platforms/website/ui-foundation.md`
- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/flow-customer-shell.md`
- `docs/invariants/website.md` (W5, W6, W19, W20)
