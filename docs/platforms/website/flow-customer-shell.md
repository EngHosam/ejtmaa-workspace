# Website Flow — Customer Main Shell (`CUSTOMER_MAIN` layout)

Customer portal contract for the customer shell (see `overview.md`).

## 1) Scope

Authed customer shell on `website/`: `CUSTOMER_MAIN` layout, identity-first header, credit footer, portaled quick-nav drawer (grid tiles), and customer `me` read boundary via `CUSTOMER_ME`.

**Shipped in this shell:** layout wiring, `CustomerHome` empty page, header, footer, drawer, `useMe` / SSR hydrate.

**Not shipped yet:** `CustomerBottomBar`, `BottomIcons`, `CustomerSubHeader`, `Dims.bottomBarHeight`, and the drawer target routes other than `CustomerHome` (tiles render disabled until registered).

## 2) Layout wiring

- `Layout` type union includes `"CUSTOMER_MAIN"` in `website/src/types/extends/global.ts`.
- `website/src/app/ui/base/core/MyApp.tsx` `getLayout()`: `case "CUSTOMER_MAIN": return CustomerMainLayout`.
- `website/src/resources/configs/routes.ts`: `CustomerHome` at `customerRouter("")` → `/customer`, `mustAuthedAs: ["CUSTOMER"]`, `layout: "CUSTOMER_MAIN"`, `preLoadedPage: () => CustomerHome`.
- `CustomerMainLayout` (`website/src/app/ui/layouts/CustomerMainLayout.tsx`, `withMemo`) composes, in order:
  1. `CustomerDrawer` (`open` / `onClose`) — layout sibling (not nested inside the header)
  2. `CustomerHeader` (`onMenuClick` toggles drawer)
  3. Content `Box` (`minH: 100vh`, `paddingTop: Dims.headerHeight`, and at `SW.max_md` → `Dims.mobileHeaderHeight`)
  4. `CustomerFooter`
- Drawer open state is **layout-owned** (`useState` + `useCallback` toggle/close).

## 3) Header — `CustomerHeader`

`website/src/app/ui/components/customer/CustomerHeader.tsx` (`withMemo`). Identity-first fixed bar: `zIndex.header`, `semanticColor.headerBackground`, `backdropFilter: blur(14px)`, 2px accent rail (`semanticColor.accentActionBackground`) on the bottom edge.

| Zone | Content |
|---|---|
| Start | Menu `HeaderIconButton` with `DrawerMenuIcon` (all breakpoints) → desktop-only `Logo` (`hideAt maxW md`, click → `nav.push({identify: "CustomerHome"})`) → identity cluster |
| Identity | `IdentityAvatar` + greeting (`header.greeting`) + display name |
| End | Notifications `HeaderIconButton` (`FiBell`); enabled only if `routes` has `CustomerNotifications`; otherwise disabled placeholder |

- Display name: `(me.name ?? "").trim()` or i18n `header.defaultName` (`العميل` / `Customer` — subscription owner actor, not an org meeting member).
- Avatar: `me.avatar_url` or `FiUser` fallback inside `IdentityAvatar`.
- No page nav links; theme/language live in the drawer utility footer.

### 3.1) Header helpers

| File | Role |
|---|---|
| `HeaderIconButton.tsx` | Square control (`semanticDims.control.iconButtonSize`); `Icon` (`IconType` + optional `flipIcon` → `Icn` `flp`) **or** `icon` ReactNode; `disabled` dims + blocks click |
| `IdentityAvatar.tsx` | Circular avatar; optional `size` (default `2.5`); `withMemo` |
| `DrawerMenuIcon.tsx` | Shared Utils mark at `components/DrawerMenuIcon.tsx` (not under `customer/`); accent rail + two ink bars; also used by `LandingHeader` |

## 4) Footer — `CustomerFooter`

`website/src/app/ui/components/customer/CustomerFooter.tsx` (`withMemo`).

- Start: `© {year} {app.title} — {footer.rights}`
- End: `MicrobandCredit compact`
- Shell: `semanticColor.footerBackground`, `br_t` footer border, `pv={2}`, `mt={3}`, `Container` + `Row jc_sb`
- Mobile bottom clearance for a future fixed bottom bar is **deferred** until `CustomerBottomBar` + `Dims.bottomBarHeight` ship.

## 5) Drawer — `CustomerDrawer`

`website/src/app/ui/components/customer/CustomerDrawer.tsx` — **no** `withMemo` (direction / open / route reactive).

### 5.1) Chrome

- Portal: `createPortal(..., document.body)` when client-mounted and visibility window open.
- Pattern matches `LandingMobileDrawer`: `Fixed` fill + blurred `Absolute` backdrop (`semanticColor.backdrop`, opc 0.48) + sliding panel via Utils `motion` (`x` off-screen uses `router.isRTL`).
- Props: `open`, `onClose`. Closing: backdrop click, close control, successful nav, logout.
- Body scroll lock: `document.body` class `no-scroll-drawer` while `open`.
- Exit animation: keep portal mounted ~220ms after `open` becomes false (`DRAWER_ANIMATION_MS`).
- Panel width: `semanticDims.shell.drawerWidth`; `zIndex.modals`.

### 5.2) Structure (top → bottom)

1. Title row: i18n `drawer.title` (**تنقل سريع** / **Quick navigation**) + close button (`FiChevronLeft` + `Icn` `flp` — points out of the panel).
2. Hero: larger `IdentityAvatar` (`size={3.4}`) + name + `roleCustomer` chip on `primaryActionBackground`.
3. Scrollable **2-column `Grid`** of large tile buttons (`DrawerGridItem`).
4. Utility footer: label `utilityPrefs` + `ThemeModeSwitch compact` + `LanguageSwitch compact` + full-width logout (`FormActionButton` tone `neutral` → `auth.logout`).

### 5.3) Grid tiles (nav)

Tiles are route-gated with `Object.prototype.hasOwnProperty.call(routes, identify)`. Absent identifies render `disabled` + opacity 0.55. `CustomerHome` is always treated available; logout is always functional.

Order (product-aligned to backend `CustomerSchema` roots / customer settings; user-approved):

| Order | i18n key | Identify | Backend surface |
|---|---|---|---|
| 1 | `itemHome` | `CustomerHome` | shell home (shipped route) |
| 2 | `itemMeetings` | `CustomerMeetings` | `meetings` / `meeting` |
| 3 | `itemMembers` | `CustomerMembers` | `members` / `member` |
| 4 | `itemOrganization` | `CustomerOrganization` | `organization` |
| 5 | `itemMessageTemplates` | `CustomerMessageTemplates` | `messageTemplates` / `messageTemplate` |
| 6 | `itemSubscription` | `CustomerSubscription` | `plans` / `subscriptions` |
| 7 | `itemSettings` | `CustomerSettings` | `CustomerRequester` read/update settings |
| 8 | `itemSupport` | `CustomerSupport` | no GQL root yet — placeholder |
| 9 | `itemHelp` | `CustomerHelpGuide` | static info (planned route) |

Tile chrome: card background, navy icon well (`primaryActionBackground`) + white glyph (`primaryActionText`). Hover (available only): icon-well background → `accentActionBackground` (orange) + `scale(1.06)`; icon stays white; no tile fill swap and no shadow.

Nav is close-first: `onClose` then `nav.push({identify})`.

## 6) Customer `me` — `useMe` + `CUSTOMER_ME`

- Adapter id: `DATA_ADAPTERS.CUSTOMER_ME` in `website/src/resources/configs/store/data-adapters.ts`.
- **Must** use `api: API.DATA_ADAPTERS.CUSTOMER.GQL` (`/data_adapters/customer/gql`) — not the visitor/global `DATA_ADAPTERS.GQL`.
- Hook: `website/src/app/ui/components/customer/hooks/useMe.tsx`.
  - Default: `useAdapter` + core query `me { name avatar_url }`, `enterMode` `LOAD_ON_MOUNT` (or `FORCE_RELOAD_ON_MOUNT` when `updateOnEnter`).
  - With `data.query`: `useShallowAdapter` keyed by `md5(data.query)`, inherits `CUSTOMER_ME`.
  - No auth `useSelector` gate inside the hook — customer routes/shell already require `CUSTOMER`.
  - Socket: `useSocket({ registerTo: "OnCustomerEvent", ... })` → `mLoad({reload: true})`.
- SSR: `auth.loadCurrentCustomer(myInstance)` in `website/src/app/services/auth.ts` (no-op unless `isAuthedAs(..., ["CUSTOMER"])`) calls `LoadCurrentCustomer`; invoked from `boot.server` in `website/src/app/services/index.ts` after start + permissions.
- Socket registry key must be `OnCustomerEvent` (mirrors backend notify event name) in `website/src/resources/configs/socket/events.ts`.

## 7) Home page

`website/src/app/ui/pages/customer/CustomerHome.tsx` — `MyPage` with `Main()` returning `null` (empty shell host). Authed customer landing via `getMyHomeIdentify` → `"CustomerHome"`.

## 8) Bottom bar — planned

`CustomerBottomBar` + `BottomIcons` and `Dims.bottomBarHeight` are **not** in the website tree yet. Prior contract target (three items: Home / Notifications / Settings) remains deferred; do not invent clearance tokens until that stage.

## 9) Theme + z-index (shipped usage)

| Token / value | Consumer |
|---|---|
| `Dims.headerHeight` / `Dims.mobileHeaderHeight` | `CustomerMainLayout` content padding; header height |
| `zIndex.header` | `CustomerHeader` |
| `zIndex.modals` | `CustomerDrawer` portal |
| `semanticDims.shell.drawerWidth` | Drawer panel |
| Accent orange | Header rail, `DrawerMenuIcon` rail, drawer tile hover well |

## 10) i18n

`website/src/resources/translations/ar.ts` + `en.ts` → `ui.layouts.customerMainLayout`:

- `header.*` — menu, greeting, defaultName, home/notifications aria
- `footer.rights`
- `drawer.*` — title (**تنقل سريع** / **Quick navigation**), close, role chip, nine tile labels, prefs, logout

Shipped tile label contract (user-approved wording):

| Key | ar | en |
|---|---|---|
| `itemHome` | الرئيسية | Home |
| `itemMeetings` | الاجتماعات | Meetings |
| `itemMembers` | الأعضاء | Members |
| `itemOrganization` | المؤسسة | Organization |
| `itemMessageTemplates` | قوالب الرسائل | Message templates |
| `itemSubscription` | الاشتراك | Subscription |
| `itemSettings` | الإعدادات | Settings |
| `itemSupport` | الدعم الفني | Support |
| `itemHelp` | المساعدة | Help |

Do not use longer forms such as «معلومات المؤسسة» or «الخطط / الاشتراك» unless product reopens the copy.

## 11) Route-reactive constraints (W24)

- `CustomerDrawer` — **without** `withMemo`.
- `CustomerMainLayout`, `CustomerHeader`, `CustomerFooter`, `HeaderIconButton`, `IdentityAvatar`, `DrawerMenuIcon` — `withMemo`.
- Future `CustomerBottomBar` / `CustomerSubHeader` — without `withMemo` when added.

See `docs/invariants/website.md` W24 and `.cursor/rules/website-route-reactive-components.mdc`.

## 12) Traceability map (this shell change set)

| Path | Doc anchor |
|---|---|
| `website/src/types/extends/global.ts` | §2 Layout wiring |
| `website/src/app/ui/base/core/MyApp.tsx` | §2 |
| `website/src/app/ui/layouts/CustomerMainLayout.tsx` | §2 |
| `website/src/app/ui/pages/customer/CustomerHome.tsx` | §7 |
| `website/src/resources/configs/routes.ts` | §2; also `route-registry-contract.md` |
| `website/src/app/services/router.ts` | §7; `route-registry-contract.md` §6 |
| `website/src/app/services/auth.ts` | §6 SSR |
| `website/src/app/services/index.ts` | §6 SSR; `ssr-boot-and-startup.md` |
| `website/src/resources/configs/axios/api.ts` | §6; `data-flow-and-gql.md` |
| `website/src/resources/configs/store/data-adapters.ts` | §6 |
| `website/src/resources/configs/socket/events.ts` | §6 |
| `website/src/app/ui/components/customer/hooks/useMe.tsx` | §6 |
| `website/src/app/ui/components/customer/CustomerHeader.tsx` | §3 |
| `website/src/app/ui/components/customer/HeaderIconButton.tsx` | §3.1 |
| `website/src/app/ui/components/customer/IdentityAvatar.tsx` | §3.1 |
| `website/src/app/ui/components/DrawerMenuIcon.tsx` | §3.1; also `landing-page.md` § Header |
| `website/src/app/ui/components/LandingHeader.tsx` | Shared mark consumer; `landing-page.md` § Header |
| `website/src/app/ui/components/customer/DrawerMenuIcon.tsx` | **Deleted** — moved to shared `components/DrawerMenuIcon.tsx` |
| `website/src/app/ui/components/customer/CustomerFooter.tsx` | §4 |
| `website/src/app/ui/components/customer/CustomerDrawer.tsx` | §5 |
| `website/src/resources/translations/ar.ts` / `en.ts` | §10 |
| `website/src/app/services/toast.tsx` | Incidental: toast position RTL/LTR top corners (not shell-specific) |
| `website/lib/tsconfig.tsbuildinfo` | Generated build info — not narrated |

## 13) Related

- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/website/shared-ui-and-shell.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/ssr-boot-and-startup.md`
- `docs/platforms/website/flow-auth.md`
- `docs/invariants/website.md` (W24, W26, W27)
- `.cursor/rules/website-customer-drawer-nav-backend-alignment.mdc`
- `.cursor/rules/website-customer-utils-composed-marks.mdc`
