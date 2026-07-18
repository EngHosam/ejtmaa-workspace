# Website Shared UI + Shell

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native shared UI primitives and shell for `website/`: document shell, actor-specific headers, side drawers, authed customer bottom bar (`CustomerBottomBar`), and shared presentational primitives -- composed from `Utils` and semantic theme tokens.

## 1.1) Shipped shell

Layouts: `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`), `CustomerMainLayout` (`CUSTOMER_MAIN`).

| Context | Components |
|---|---|
| Auth + error (`BASIC`) | `ThemeModeSwitch`, `LanguageSwitch` in `BasicLayout` top row; auth pages use `AuthPageShell` |
| Landing (`LANDING`) | `LandingHeader`, `LandingFooter`, `LandingMobileDrawer`, `home/*` sections |
| Main workspace (`MAIN`) | `Header`, `Drawer`, `Footer` via `MainLayout` |
| Customer workspace (`CUSTOMER_MAIN`) | `CustomerDrawer` + `CustomerHeader` + content + `CustomerFooter` (see `flow-customer-shell.md`) |

Generic shell components: `Header`, `Drawer`, `Footer`, `Logo`, `ThemeModeSwitch`, `LanguageSwitch` (under `src/app/ui/components/`).

Main drawer (`src/app/ui/layouts/main-layout/drawer.ts`): one `business` section with `dashboard` (→ `Home`) + `logout` only.

`AuthedAs = "CUSTOMER"`; auth reducer exposes `auth.customer.permissions`.

Shared marks under `src/app/ui/components/`: `DrawerMenuIcon` (header + landing menu), `HomeMark` (breadcrumb root + drawer home tile; tones `ink` / `onPrimary`). Shared breadcrumb under `src/app/ui/components/`: `Breadcrumb` + `useBreadcrumbs` (**not** under `ui/base/`). Shared list-lane primitives: `ResultLane`, `CardSkeleton`, `LoadMoreButton`, `SearchField`, `SectionHeading`, `Wrong`. Customer helpers under `customer/`: `HeaderIconButton`, `IdentityAvatar`, `CustomerSubHeader`, `hooks/useMe`, members directory (`hooks/useCustomerMembers`, `members/*` — `flow-customer-members.md`). **Planned:** `CustomerBottomBar`, `BottomIcons`.

## 2) Document shell

- Document shell owned by `MyHtml` + `MyApp`/`MyPage` lifecycle.
- `MyHtml` owns `<html>` / `<head>` / `<body>`, theme mode, `lang` + `dir`, fonts, and global CSS.
- Safe-area insets use CSS `env(safe-area-inset-*)` or theme padding tokens.
- Authed customer shell uses `Dims.bottomBarHeight` for bottom clearance. See `flow-customer-shell.md`.

## 3) Actor-specific headers

Shipped: generic `Header` (`src/app/ui/components/Header.tsx`) with `main`/`compact` variants; visitor `LandingHeader`; customer `CustomerHeader` (identity-first — `flow-customer-shell.md` §3).

| Actor | Component | Status |
|---|---|---|
| Visitor | `LandingHeader` | shipped (landing); `LandingSubHeader` planned |
| Customer | `CustomerHeader` | shipped on `CUSTOMER_MAIN` |
| Customer | `CustomerSubHeader` | shipped when route has `breadcrumb` |

Shared header behavior (both components):

- Back behavior: `onBack` if provided, else `useNav().back()` / browser history back.
- Badge behavior: numeric badge (`1..99`, capped `99`), dot badge (`true`), none (`0`/missing).
- LTR-first authoring; runtime `dir` handles mirroring.
- Fixed dimensions from `theme.ts`/`utils.ts` tokens. `zIndex.header` above body content.
- Brand slot: the logo (`Logo preset="header"`) renders bare — no border/background/padding pill around it. The `main` variant places `resolvedBrandSlot` directly in the trailing row. See `brand-identity-alignment.md` § Logo and `.cursor/rules/website-logo-no-frame.mdc`.
- Trailing control cluster: `ThemeModeSwitch` + `LanguageSwitch` render in the trailing row for the `main` non-compact variant (gated by `showThemeSwitch`). Header icon buttons (`HeaderIconButton`: menu / back / `trailingAction` e.g. notifications) use `crn={semanticDims.card.radius}` with `semanticColor.inputBackground` + `inputBorder` + `iconPrimary` — same corner and surface tokens as the two toggles.

## 4) Side drawers

Shipped:

- Generic `Drawer` + `main-layout/drawer.ts` (dashboard + logout) for `MAIN`.
- Visitor mobile: `LandingMobileDrawer` (portaled). See landing layout docs.
- Customer: `CustomerDrawer` — portaled quick-nav grid, layout-owned open state, hero from `useMe()`, theme/language + logout in utility footer. Canonical contract: `flow-customer-shell.md` §5.

Customer drawer differs from the generic `Drawer` hero (no centered logo hero; identity row + 2-col grid tiles). Nav tiles must stay aligned to backend customer roots (`.cursor/rules/website-customer-drawer-nav-backend-alignment.mdc`).

## 5) Shared primitives

- **Button**: `Utils`-composed; `actionPrimary` solid navy (`semanticColor.primaryActionBackground`) for primary CTAs; RTL-safe arrow mirroring via CSS `dir`; loading state via `Loadable` overlay.
- **SegmentedPills**: tab/scope semantics; a11y label per option.
- **Loadable**: spinner/progress surface for initial loading, submit-busy, and overlay loaders.
- **Toast**: success/error notifications; translation-backed copy.
- **Empty / LaneFailed** (`Wrong.tsx`): absolute overlays for ResultLane; i18n `ui.components.wrong.*`. Guest / Able remain planned/target where not present.
- **ResultLane**: responsive card grid + skeleton + load-more + empty/failed; consumer supplies `renderCard`. Current `CardSkeleton` is member-card shaped — see `flow-customer-members.md` §6 and `.cursor/rules/website-result-lane-skeleton-shape.mdc`.
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
