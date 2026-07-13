# CPanel Main Layout Responsive Shell

## Purpose

Responsive-shell contract for the supervisor control-panel `MAIN` layout.
It exists so future shell work can preserve the documented behavior, debug mobile drawer issues, and extend the layout without reintroducing duplicated desktop/mobile shell trees.

## Scope

The contract covers these files when present in checkout:

- `cpanel/src/app/ui/layouts/MainLayout.tsx`
- `cpanel/src/app/ui/components/Header.tsx`
- `cpanel/src/app/ui/components/Footer.tsx`
- `cpanel/src/app/ui/components/MicrobandCredit.tsx`
- `cpanel/src/resources/emotion/styles/global.ts`

## Entry Point

`cpanel/src/app/ui/layouts/MainLayout.tsx` remains the single shared route shell for all `MAIN` pages selected by `MyApp`.

Contract truth:

- the layout renders one shared shell tree with CSS-controlled desktop/mobile presentation,
- the desktop drawer rail and the mobile drawer overlay share layout-owned drawer data and shell composition,
- the responsive breakpoint aligns to `SW.min_lg` from `cpanel/src/resources/configs/theme.ts` (when present in checkout),
- desktop-versus-mobile presentation is controlled by CSS media queries in layout and shell components.

## Responsive Contract

### 1. Single shell tree

`MainLayout` keeps one shared structure:

1. optional mobile overlay layer,
2. shell padding wrapper,
3. shell row,
4. desktop drawer rail container,
5. content column with `Header`, page surface, and `Footer`.

The desktop drawer rail is hidden on smaller breakpoints by CSS.
The mobile drawer overlay is hidden on desktop breakpoints by CSS.
This prevents the previous "desktop shell first, mobile shell later" duplication pattern.

### 2. CSS-first responsive ownership

The layout treats responsiveness as a styling concern:

- shell padding swaps between desktop and mobile values through media queries,
- shell gaps swap through media queries,
- page surface minimum height swaps through media queries,
- the header and footer compact presentation is driven by responsive CSS hooks instead of layout-level viewport state,
- the drawer body-scroll lock applies on mobile widths only in `global.ts`.

JS is still used for interactive state only:

- desktop drawer collapsed/expanded state,
- mobile drawer open/close animation state,
- navigation/logout actions.

## Header and Footer Adaptation

### Header

`cpanel/src/app/ui/components/Header.tsx` exposes responsive shell-specific hooks for the `MAIN` layout:

- `responsiveCompact`
- `responsiveBreakpoint`
- `onDesktopMenuClick`
- `onMobileMenuClick`

Behavior:

- the desktop menu button remains mounted but is hidden below the responsive breakpoint,
- a mobile menu button is shown only below the responsive breakpoint,
- title/subtitle, theme switch, and trailing action stay visible on desktop widths and are hidden on the compact mobile presentation,
- brand pill padding also compacts below the responsive breakpoint.

### Footer and credit

`cpanel/src/app/ui/components/Footer.tsx` and `cpanel/src/app/ui/components/MicrobandCredit.tsx` expose matching responsive-compact hooks so the shared shell footer compacts through CSS without layout-owned viewport state.

Behavior:

- footer gap, padding, alignment, and brand column flex behavior compact below the responsive breakpoint,
- the Microband credit pill reduces its padding and logo size below the responsive breakpoint.

## Drawer Overlay and Scroll Lock

The mobile drawer overlay still uses layout-owned React state for open/close animation timing.

Contract truth:

- `mobileDrawerOpen` controls whether the drawer is interactable,
- `mobileDrawerVisible` keeps the overlay mounted long enough for the close animation to complete,
- the `no-scroll-drawer` body class is still added while the mobile drawer is open,
- `cpanel/src/resources/emotion/styles/global.ts` limits the `overflow: hidden` effect for `no-scroll-drawer` to mobile widths only.

## Force Mode Contract

`MainLayout` still accepts:

- `forceMode="desktop"`
- `forceMode="mobile"`

Purpose:

- shell review states,
- embedded/mockup rendering,
- deterministic visual testing or local inspection.

When `forceMode` is provided, the layout bypasses the responsive media-query branch selection for that specific shell concern and uses the forced presentation directly.

## Failure and Debug Notes

If the mobile drawer appears correct visually but page scrolling is unexpectedly locked, inspect:

1. whether `body` still has class `no-scroll-drawer`,
2. whether the viewport is below the `SW.min_lg` breakpoint,
3. whether the mobile drawer close animation completed and cleared `mobileDrawerVisible`.

If a future change reintroduces duplicated desktop/mobile shell trees, treat that as a regression against the shell contract documented here.

## Contract inventory

| Path | Status | Described in |
|------|--------|--------------|
| `cpanel/src/app/ui/layouts/MainLayout.tsx` | Behaviorally documented | `Entry Point`, `Responsive Contract`, `Drawer Overlay and Scroll Lock`, `Force Mode Contract` |
| `cpanel/src/app/ui/components/Header.tsx` | Behaviorally documented | `Header and Footer Adaptation` -> `Header` |
| `cpanel/src/app/ui/components/Footer.tsx` | Behaviorally documented | `Header and Footer Adaptation` -> `Footer and credit` |
| `cpanel/src/app/ui/components/MicrobandCredit.tsx` | Behaviorally documented | `Header and Footer Adaptation` -> `Footer and credit` |
| `cpanel/src/resources/emotion/styles/global.ts` | Behaviorally documented | `Drawer Overlay and Scroll Lock` |
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated/runtime metadata | Not narrated line-by-line; modified by local TypeScript verification output rather than product behavior |
| `docs/platforms/cpanel/main-layout-responsive-shell.md` | Responsive shell contract page | Entire document |
| `docs/platforms/cpanel/overview.md` | Platform overview for shell routes and layout | shell routes and layout sections |
| `docs/platforms/cpanel/repository-inventory.md` | Repository inventory for shell-related paths | `6) src/resources non-config files`, `12) src/app/ui/components`, `13) src/app/ui/layouts` |
| `docs/platforms/cpanel/ui-foundation.md` | UI foundation for Utils and theme contract | Utils + theme contract |
| `.cursor/rules/cpanel-platform-governance.mdc` | Durable cpanel governance constraints | `Cpanel constraints` |
| `.cursor/skills/cpanel-platform-governance/SKILL.md` | Repeatable cpanel workflow guidance | `UI Rules` |

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/cpanel/repository-inventory.md`
- `docs/platforms/cpanel/ui-foundation.md`
- `.cursor/rules/cpanel-platform-governance.mdc`
