# Website Flow — Customer Main Shell (`CUSTOMER_MAIN` layout)

Customer portal contract for the customer shell (see `overview.md`).

## 1) Scope

Authed customer shell on `website/`: the `CUSTOMER_MAIN` layout, header/footer/drawer/bottom-bar composition, customer `me` read boundary, and custom bottom-bar icons.

## 2) Layout wiring

- `Layout` type union includes `"CUSTOMER_MAIN"` in `website/src/types/extends/global.ts`.
- `website/src/app/ui/base/core/MyApp.tsx` `getLayout()` switch: `case "CUSTOMER_MAIN": return CustomerMainLayout`.
- `website/src/resources/configs/routes.ts`: `CustomerHome` (`/customer`, `mustAuthedAs: ["CUSTOMER"]`) uses `layout: "CUSTOMER_MAIN"`. Every customer page that carries this shell sets the same layout.
- `CustomerMainLayout` (`website/src/app/ui/layouts/CustomerMainLayout.tsx`) composes: `CustomerHeader` (fixed) → content `Box` (`min-h: 100vh`, `paddingTop: Dims.headerHeight`/`mobileHeaderHeight`, `paddingBottom: calc(Dims.bottomBarHeight + 4rem)`) → `CustomerFooter` → `CustomerBottomBar`.

## 3) Header — `CustomerHeader`

`website/src/app/ui/components/customer/CustomerHeader.tsx`. Fixed, blurred shell (`backdropFilter: blur(14px)`, `zIndex.header`, `semanticColor.headerBackground`). Left: `CustomerDrawer` + `Logo` (click → `nav.push({identify: "CustomerHome"})`). Right: notifications bell (`FiBell`).

- Notifications bell targets `CustomerNotifications`. Availability follows `Object.prototype.hasOwnProperty.call(routes, "CustomerNotifications")`; when the identify is absent the bell renders as a disabled placeholder (`disabled`, `aria-disabled`, `opacity 0.55`, `cursor: default`).
- `CustomerHeader` reads the static `routes` registry only, so it is exported with `withMemo`.

## 4) Footer — `CustomerFooter`

`website/src/app/ui/components/customer/CustomerFooter.tsx`. Bottom credit strip: `© {year} {appTitle} — {rights}` on the start side, `MicrobandCredit` on the end side.

- Mobile-only bottom clearance: `marginBottom: Dims.bottomBarHeight` inside `@media (max-width: SW.max_md)` so the fixed `CustomerBottomBar` does not cover the footer.

## 5) Drawer — `CustomerDrawer`

`website/src/app/ui/components/customer/CustomerDrawer.tsx`. Portaled (`createPortal` to `document.body`) + `framer-motion` slide-in side drawer. RTL-aware side: LTR slides from the left, RTL from the right.

- Drawer sections: activity items (`CustomerNotifications`), info items (`CustomerHelpGuide`, `CustomerAbout`, `CustomerTerms`, `CustomerSettings`), utility footer (`LanguageToggle`, `ThemeModeSwitch`, `LogoutRow`).
- Each nav row shows a trailing `FiChevronRight` (flipped via `flp` for RTL). Route-gated items check `routes` registry membership; absent identifies render as disabled placeholders. `CustomerHome` and `Logout` are always functional.
- Hero identity binds to `useMe()`: avatar (`me.avatar_url`) with `FiUser` fallback, name (`me.name` → `heroDefaultName` fallback), `roleCustomer` chip.
- `CustomerDrawer` reads `isRTL` at render time — exported **without** `withMemo`.

## 6) Customer `me` — `useMe` hook + `CUSTOMER_ME` adapter

- `website/src/resources/configs/store/data-adapters.ts`: `DATA_ADAPTERS.CUSTOMER_ME = "CUSTOMER_ME"` with `initDataAdaptersProps[CUSTOMER_ME] = {api: API.DATA_ADAPTERS.CUSTOMER.GQL}`.
- `website/src/app/ui/components/customer/hooks/useMe.tsx`: customer profile hook. With `data.query` → `useShallowAdapter` keyed by `md5(data.query)` inheriting `CUSTOMER_ME`; without args → `useAdapter({adapterIdentify: CUSTOMER_ME, enterOptions: {query: coreQuery}, enterMode})` where `coreQuery` is `query Query { me { name avatar_url } }`. Auth state reads `state.auth` via `useSelector`. Refreshed on `OnCustomerEvent` via `useSocket`.
- Socket mirror: `OnCustomerEvent` with payload `UPDATED` per `socket-event-mirroring.md`.

## 7) Bottom bar — `CustomerBottomBar` + `BottomIcons`

- Three nav items: `CustomerHome`, `CustomerNotifications`, `CustomerSettings`.

- Mobile: `position: fixed` at bottom, full-width; nav row height = `Dims.bottomBarHeight`. Desktop: floating `30rem` pill with `1rem` bottom margin.
- Icons: `BottomHomeIcon` (custom SVG); notifications and settings use Feather (`FiBell`, `FiSettings`).
- Nav targets follow `routes` registry membership via `hasOwnProperty`; absent identifies render disabled + muted (`opacity 0.6`, `aria-disabled`). Active item color = `iconPrimary`, inactive = `iconSecondary`.
- `CustomerBottomBar` reads `currentPage.identify` and `isRTL` — exported **without** `withMemo`.

## 8) Custom bottom-bar glyphs — `BottomIcons`

`website/src/app/ui/components/customer/BottomIcons.tsx`. `BottomHomeIcon` as a `React.forwardRef` SVG component taking `size` + `color`.

## 9) Theme + spacing tokens

- `website/src/resources/configs/theme.ts`: `Dims.bottomBarHeight = "4rem"` — consumed by `CustomerBottomBar`, `CustomerMainLayout`, and `CustomerFooter`.
- Bottom clearance uses `Dims.bottomBarHeight` via margin (footer) / padding (content).
- `zIndex.menu` for bottom bar, `zIndex.header` for header, `zIndex.modals` for drawer portal.
- Deck hosts with high intra-deck `zIndex` use `isolation: isolate` (W40, `.cursor/rules/website-deck-stacking-isolation.mdc`).

## 10) i18n

`website/src/resources/translations/ar.ts` + `en.ts`: `ui.layouts.customerMainLayout` namespace with `header`, `footer`, `drawer`, `bottomBar` keys.

## 11) Route-reactive constraints (W24)

- `CustomerDrawer` and `CustomerBottomBar` — **without** `withMemo`.
- `CustomerSubHeader` — **without** `withMemo`.
- `CustomerMainLayout` — `withMemo`; sub-header from `routes[identify]?.breadcrumb`.
- `CustomerHeader` and `CustomerFooter` — `withMemo`.

See `docs/invariants/website.md` W24 and `.cursor/rules/website-route-reactive-components.mdc`.

## 12) Related

- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/website/shared-ui-and-shell.md`
- `docs/platforms/website/flow-auth.md`
- `docs/invariants/website.md` (W24, W26, W27, W40)
