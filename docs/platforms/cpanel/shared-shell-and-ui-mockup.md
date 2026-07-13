# CPanel Shared Shell and UI Mockup

## Purpose

Supervisor cpanel shared-shell contract for `MainLayout` drawer, footer, and review-route parity.

This document describes the shared shell contract for the supervisor cpanel. The shell includes:

- a layout-owned drawer data source under `src/app/ui/layouts/main-layout/`,
- a regrouped shared drawer with desktop and mobile behavior refinements,
- a simplified footer with dedicated `MicrobandCredit`,
- Arabic shell copy and review-route parity,
- a mobile overlay with body-scroll locking and blur,
- a fixed-height drawer shell with internal scrolling,
- a desktop-only custom scrollbar treatment for the existing drawer sections viewport,
- and the supporting static Microband asset under `cpanel/public/images/`.

This page supports implementation, review, debugging, and follow-up shell work.
It documents the contract outcomes the shell files must satisfy when present in checkout.

## Shell surface

The cpanel shell and review route (`UiMockup`) share `MainLayout` drawer/footer composition documented here.

Contract outcomes:

- `MainLayout` owns `Header` + `Drawer` + `Footer`; drawer/navigation content comes from a layout-owned helper module.
- Shared drawer groups three translated sections: business, operations, and settings, with logout as a utility action.
- Desktop and mobile drawer panels use explicit panel heights from viewport height minus outer shell padding; sections area owns internal scrolling.
- Desktop sections viewport uses a local custom scrollbar treatment.
- Mobile drawer locks body scroll while open and renders a blurred, semi-transparent overlay behind the panel.
- Drawer hero shows `مرحبا بك`, omits descriptive subtitle, and keeps compact theme switch in mobile contexts.
- Shared footer renders shell title plus `MicrobandCredit` component and logo asset.
- `UiMockup` consumes the same drawer-builder helpers and footer contract as `MainLayout`.

## Route and Layout Contract

### Route mapping

`cpanel/src/resources/configs/routes.ts` maps:

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `MAIN` |
| `Customers` | `/customers` | `MAIN` |
| `Customer` | `/customer/:formType(show)/:id` | `MAIN` |
| `AccountSettings` | `/account-settings` | `MAIN` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `Error` | `/:error(404\|500\|403)` | `BASIC` |

`UiMockup` remains a lazily loaded review route.

### layout resolution

`cpanel/src/app/ui/base/core/MyApp.tsx` still resolves:

- `MAIN` -> `MainLayout`
- any other value -> `BasicLayout`

The shell contract applies to every route that renders on `MAIN`, including the review route.

## Shared Shell Composition

### Layout-owned drawer data source

`cpanel/src/app/ui/layouts/main-layout/drawer.ts` owns the reusable drawer section and utility-item definitions for the `MAIN` shell.

Behavior:

- defines the translated section keys `business`, `operations`, and `settings`,
- defines the translated item keys used by the shared shell,
- binds concrete `react-icons/fi` icons to each item,
- marks `dashboard` as the active item inside the review route drawer data,
- exports `buildMainLayoutDrawerSections(...)` and `buildMainLayoutDrawerUtilityItems(...)`,
- is consumed by both `MainLayout` and `UiMockup`.

This keeps the layout-owned navigation structure under `layouts/` rather than burying it in a generic component or duplicating it in the review route.

### Drawer

`cpanel/src/app/ui/components/Drawer.tsx` is the shared drawer surface.

Behavior:

- supports desktop and mobile modes through the `mode` prop,
- uses `panelHeight` to keep the visible panel at a fixed shell height,
- renders the hero area, a scrollable sections viewport, and utility actions in one panel,
- embeds the compact `ThemeModeSwitch` only when `showThemeSwitch` resolves to `true` (mobile by default, desktop disabled by the `Drawer` caller),
- keeps the utility area pinned after the scrollable sections viewport by using `marginTop: auto`,
- applies the custom scrollbar styling only when `desktopCustomScrollbar` is enabled,
- leaves mobile on the same internal sections viewport but without the desktop-only custom scrollbar styling.

Desktop scrollbar behavior:

- the existing sections viewport remains the only scroll owner,
- Firefox gets `scrollbarWidth: "thin"` plus a thumb/track color pair,
- WebKit gets a thin `0.45rem` scrollbar width,
- the track remains visually transparent,
- the thumb uses a quiet neutral gradient with a small border and a darker hover state.

The viewport remains the scroll owner; there is no parallel overlay scrollbar implementation.

### Footer and Microband credit

`cpanel/src/app/ui/components/Footer.tsx` is the shared shell footer contract.

Behavior:

- supports default and compact variants,
- renders only the resolved shell brand slot plus the footer title and `MicrobandCredit`,
- uses `semanticColor.textTertiary` for the footer title to keep the shell title intentionally subdued,
- subtitle, link, and meta-text props are omitted from the shared footer contract.

`cpanel/src/app/ui/components/MicrobandCredit.tsx` is the dedicated credit surface used by the footer.

Behavior:

- renders an external link to `https://microband.com.sa`,
- uses the translated Arabic lead text and brand text from the central translation file,
- renders `public/images/microband.png`,
- removes default link underlines through local button-reset styling,
- supports compact spacing through the `compact` prop.

### MainLayout

`cpanel/src/app/ui/layouts/MainLayout.tsx` is the shell orchestrator.

Desktop behavior:

- computes `desktopDrawerPanelHeight` as `calc(100vh - (page.padY * 2))`,
- passes that fixed panel height into `Drawer`,
- keeps the main content surface at a minimum height derived from viewport height minus page padding, header height, and shell gap,
- preserves content growth because the page surface uses `minH` rather than a hard flex lock.

Mobile behavior:

- computes `mobileDrawerPanelHeight` using the same viewport-minus-padding rule,
- opens the drawer inside an absolute overlay layer,
- keeps overlay presence separate from panel open state through `mobileDrawerVisible`,
- closes the drawer when the user clicks outside the panel,
- adds `no-scroll-drawer` to `body` while the mobile drawer is open and removes it on close/unmount,
- uses a blurred overlay (`blur(8px)`) with `0.48` opacity behind the mobile panel,
- uses RTL-aware slide directions through `useRouter().isRTL`.

### Global shell styling dependency

`cpanel/src/resources/emotion/styles/global.ts` is part of the mobile-drawer runtime behavior, including document resets.

Relevant behavior:

- still resets `html` / `body` margins and padding,
- still flips icons in RTL when they are not marked `no-flip`,
- `cpanel/src/resources/emotion/styles/global.ts` limits the `overflow: hidden` effect for `no-scroll-drawer` to mobile widths only, so a stale class does not lock desktop scrolling.

## Review Route Parity

`cpanel/src/app/ui/pages/UiMockup.tsx` is the review route for the shared shell and components.

Shell-review behavior:

- imports the same drawer builder helpers as `MainLayout`,
- renders desktop and mobile shell previews through `MainLayout embedded` forced modes,
- renders standalone drawer previews using the same title/subtitle and grouped data source used by the live shell,
- renders footer previews using the simplified title-only footer contract,
- continues to provide review-only content for `ThemeModeSwitch`, `Header`, `Loadable`, `Toast`, `Footer`, `Drawer`, and `DataTable`.

The route is review-oriented rather than a business feature screen; shell preview data matches the live shell contract.

## Copy and Assets

### Translations

`cpanel/src/resources/translations/ar.ts` defines the following shell-copy contract:

- `mainLayout.drawerTitle` is `مرحبا بك`,
- `mainLayout.drawerSubtitle` is intentionally empty,
- the shared drawer section labels are `business`, `operations`, and `settings`,
- `microbandCredit.lead` and `microbandCredit.brand` are dedicated shared footer-credit strings,
- `UiMockup.drawerReview` and `UiMockup.footerReview` mirror the same shared shell wording.

### Static assets

The shell asset set is:

- `cpanel/public/images/light_logo.png`
- `cpanel/public/images/dark_logo.png`
- `cpanel/public/images/microband.png`

All three are binary assets and are therefore accounted for here by runtime purpose rather than line-by-line narrative.

## Generated Artifact Accounting

`cpanel/lib/tsconfig.tsbuildinfo` is a generated incremental TypeScript artifact, not a source-of-truth implementation file.
It is accounted for in the contract inventory only and is not narrated as a behavioral contract.

## Contract inventory

The table below inventories every tracked implementation path for the shared shell contract.

| Path | Contract role | Where documented |
|------|--------------------------------|------------------|
| `cpanel/src/app/ui/layouts/main-layout/drawer.ts` | Layout-owned source of grouped drawer sections and utility items | `# Shared Shell Composition` / `### Layout-owned drawer data source` |
| `cpanel/src/app/ui/components/Drawer.tsx` | Shared drawer surface with grouped sections, internal scrolling, desktop custom scrollbar styling, and mobile/desktop panel modes | `# Shared Shell Composition` / `### Drawer` |
| `cpanel/src/app/ui/components/Footer.tsx` | Simplified shared footer contract reduced to title plus Microband credit | `# Shared Shell Composition` / `### Footer and Microband credit` |
| `cpanel/src/app/ui/components/MicrobandCredit.tsx` | Dedicated linked Microband credit component used by the footer | `# Shared Shell Composition` / `### Footer and Microband credit` |
| `cpanel/src/app/ui/layouts/MainLayout.tsx` | Shell orchestrator for viewport-based drawer heights, mobile body-locking, blurred overlay, and min-height content surfaces | `# Shared Shell Composition` / `### MainLayout` |
| `cpanel/src/app/ui/pages/UiMockup.tsx` | Review route consumes the same drawer builders and footer contract as the live shell | `# Review Route Parity` |
| `cpanel/src/resources/translations/ar.ts` | Centralized Arabic shell wording for grouped navigation, `مرحبا بك`, empty drawer subtitle, footer review title, and Microband credit copy | `# Copy and Assets` / `### Translations` |
| `cpanel/src/resources/emotion/styles/global.ts` | Global body-lock support for the mobile drawer and existing document reset behavior | `# Shared Shell Composition` / `### Global shell styling dependency` |
| `cpanel/public/images/microband.png` | Binary Microband logo asset used by `MicrobandCredit` | `# Copy and Assets` / `### Static assets` |
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated incremental TypeScript artifact modified as a side effect of local tooling | `# Generated Artifact Accounting` |

