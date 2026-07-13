# Website Shared UI + Shell

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native shared UI primitives and shell for `website/`: document shell, actor-specific headers, side drawers, authed customer bottom bar (`CustomerBottomBar`), and shared presentational primitives -- composed from `Utils` and semantic theme tokens.

## 2) Document shell

- Document shell owned by `MyHtml` + `MyApp`/`MyPage` lifecycle.
- `MyHtml` owns `<html>` / `<head>` / `<body>`, theme mode, `lang` + `dir`, fonts, and global CSS.
- Safe-area insets use CSS `env(safe-area-inset-*)` or theme padding tokens.
- Authed customer shell uses `Dims.bottomBarHeight` for bottom clearance. See `flow-customer-shell.md`.

## 3) Actor-specific headers

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

## 4) Side drawer

- Visitor drawer: `components/LandingDrawer.tsx` (mobile, portaled). See `component-structure.md` and visitor shell pages.
- Authed customer drawer: `CustomerDrawer.tsx` -- portaled side drawer with route-gated nav items. See `flow-customer-shell.md` section 5.
- Role resolution from the `auth` Redux slice `authedAs` (`CUSTOMER`).
- Profile payload from `useMe()`: `name`, `avatar_url`.
- UX order: hero identity zone -> action cluster -> navigation blocks -> utility footer (language, theme, sign-out). Footer stays outside scroll body.
- Motion via `framer-motion`. Drawer actions are close-first.
- Brand slot in the hero identity zone: `Logo preset="drawer"` rendered bare inside `<Box as_fs>` (alignment only, no frame). See `brand-identity-alignment.md` § Logo.

## 5) Shared primitives

- **Button**: `Utils`-composed; `actionPrimary` solid navy (`semanticColor.primaryActionBackground`) for primary CTAs; RTL-safe arrow mirroring via CSS `dir`; loading state via `Loadable` overlay.
- **SegmentedPills**: tab/scope semantics; a11y label per option.
- **Loadable**: spinner/progress surface for initial loading, submit-busy, and overlay loaders.
- **Toast**: success/error notifications; translation-backed copy.
- **Empty / Guest / LaneFailed / Able**: shared `Utils`-built primitives with stacking rules documented in `ui-foundation.md`.
- **LanguageSegmentedBar** / **ThemeModeSegmentedBar**: drawer footer controls.

## 6) Navigation stabilization

- Web navigation is URL-router-based (`useNav` + browser history) inside `MyPage` lifecycle.
- Tab-root customer pages: header menu opens `CustomerDrawer`. Detail/subpage routes keep back-only behavior.
- Drawer actions are close-first.

## 7) Related

- `docs/platforms/website/ui-foundation.md`
- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/flow-customer-shell.md`
- `docs/invariants/website.md` (W5, W6, W19, W20)
