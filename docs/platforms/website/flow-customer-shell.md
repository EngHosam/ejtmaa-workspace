# Website Flow — Customer Main Shell (`CUSTOMER_MAIN` layout)

Customer portal contract for the customer shell (see `overview.md`).

## 1) Scope

Authed customer shell on `website/`: `CUSTOMER_MAIN` layout, identity-first header, credit footer, portaled quick-nav drawer (grid tiles), and customer `me` read boundary via `CUSTOMER_ME`.

**Shipped in this shell:** layout wiring, `CustomerHome` command map, `CustomerMembers` + form, `CustomerMeetings` + create/details, `CustomerOrganization`, `CustomerMessageChannels` + form, `CustomerMessageTemplates` + form, header, footer, drawer, `CustomerSubHeader` (breadcrumb bar), `useMe` / SSR hydrate.

**Not shipped yet:** `CustomerBottomBar`, `BottomIcons`, `Dims.bottomBarHeight`, and remaining drawer targets without routes (`CustomerSubscription`, `CustomerSettings`, `CustomerSupport`, `CustomerHelpGuide`, notifications page if still gated).

## 2) Layout wiring

- `Layout` type union includes `"CUSTOMER_MAIN"` in `website/src/types/extends/global.ts`.
- `website/src/app/ui/base/core/MyApp.tsx` `getLayout()`: `case "CUSTOMER_MAIN": return CustomerMainLayout`.
- `website/src/resources/configs/routes.ts`: `CustomerHome` at `customerRouter("")` → `/customer`, `mustAuthedAs: ["CUSTOMER"]`, `layout: "CUSTOMER_MAIN"`, `preLoadedPage: () => CustomerHome`.
- `CustomerMainLayout` (`website/src/app/ui/layouts/CustomerMainLayout.tsx`, `withMemo`) composes, in order:
  1. `CustomerDrawer` (`open` / `onClose`) — layout sibling (not nested inside the header)
  2. `CustomerHeader` (`onMenuClick` toggles drawer)
  3. Optional fixed `CustomerSubHeader` when `routes[identify]?.breadcrumb` is set
  4. Content `Box` (`minH: 100vh`; `paddingTop` = header height, or `calc(header + CUSTOMER_SUBHEADER_BAR)` on subpages; mobile uses `Dims.mobileHeaderHeight`)
  5. `CustomerFooter`
- Drawer open state is **layout-owned** (`useState` + `useCallback` toggle/close).
- `CustomerSubHeader` is **fixed** under the header (`CUSTOMER_SUBHEADER_BAR = 2.75rem`, `headerBackground` + blur). **No** `withMemo` on `CustomerSubHeader`. `Breadcrumb` may use `withMemo` (crumbs are props).

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
2. Hero: larger `IdentityAvatar` (`size={3.4}`) + customer name + workspace chip (`me.organization.name`, else drawer `workspaceFallback` / منصة اجتماع) on `accentActionBackground` / `accentActionText`.
3. Scrollable **2-column `Grid`** of large tile buttons (`DrawerGridItem`).
4. Utility footer: label `utilityPrefs` + `ThemeModeSwitch compact` + `LanguageSwitch compact` + full-width logout (`FormActionButton` tone `neutral` → `auth.logout`).

### 5.3) Grid tiles (nav)

Tiles are route-gated with `Object.prototype.hasOwnProperty.call(routes, identify)`. Absent identifies render `disabled` + opacity 0.55. `CustomerHome` is always treated available; logout is always functional.

`DrawerGridItem` accepts either Feather `Icon` (`IconType`) **or** `mark` (`ReactNode`) — same dual branch pattern as `HeaderIconButton`.

Order (product-aligned to backend `CustomerSchema` roots / customer settings; user-approved):

| Order | i18n key | Identify | Glyph | Backend surface |
|---|---|---|---|---|
| 1 | `itemHome` | `CustomerHome` | `HomeMark` `tone="onPrimary"` `size={1.45}` | shell home (shipped route) |
| 2 | `itemMeetings` | `CustomerMeetings` | `FiCalendar` | `meetings` / `meeting` |
| 3 | `itemMembers` | `CustomerMembers` | `FiUsers` | `members` / `member` (shipped directory — `flow-customer-members.md`) |
| 4 | `itemOrganization` | `CustomerOrganization` | `FiBriefcase` | `organization` (shipped settings form — `flow-customer-organization.md`) |
| 5 | `itemMessageTemplates` | `CustomerMessageTemplates` | `FiMessageSquare` | `messageTemplates` / `messageTemplate` (shipped — `flow-customer-message-templates.md`) |
| 6 | `itemSubscription` | `CustomerSubscription` | `FiCreditCard` | `plans` / `subscriptions` |
| 7 | `itemMessageChannels` | `CustomerMessageChannels` | `FiSend` | `messageChannels` / `messageChannel` (shipped directory — `flow-customer-message-channels.md`) |
| 8 | `itemSettings` | `CustomerSettings` | `FiSettings` | `CustomerRequester` read/update settings |
| 9 | `itemSupport` | `CustomerSupport` | `FiLifeBuoy` | no GQL root yet — placeholder |
| 10 | `itemHelp` | `CustomerHelpGuide` | `FiBookOpen` | static info (planned route) |

Tile chrome: card background, navy icon well (`primaryActionBackground`) + white Feather glyph (`primaryActionText`) **or** `HomeMark` on-primary tone. Hover (available only): icon-well background → `accentActionBackground` (orange) + `scale(1.06)`; no tile fill swap and no shadow.

Nav is close-first: `onClose` then `nav.push({identify})`.

## 6) Customer `me` — `useMe` + `CUSTOMER_ME`

- Adapter id: `DATA_ADAPTERS.CUSTOMER_ME` in `website/src/resources/configs/store/data-adapters.ts`.
- **Must** use `api: API.DATA_ADAPTERS.CUSTOMER.GQL` (`/data_adapters/customer/gql`) — not the visitor/global `DATA_ADAPTERS.GQL`.
- Hook: `website/src/app/ui/components/customer/hooks/useMe.tsx`.
  - Default: `useAdapter` + core query selecting identity, `organization` (id/name/logo_url/subdomain), and `currentSubscription` snapshot + nested `plan { id name }`, `enterMode` `LOAD_ON_MOUNT` (or `FORCE_RELOAD_ON_MOUNT` when `updateOnEnter`). Full selection: `useMe.tsx` `coreQuery`.
  - With `data.query`: `useShallowAdapter` keyed by `md5(data.query)`, inherits `CUSTOMER_ME`.
  - No auth `useSelector` gate inside the hook — customer routes/shell already require `CUSTOMER`.
  - Socket: `useSocket({ registerTo: "OnCustomerEvent", ... })` → `mLoad({reload: true})`.
- SSR: `auth.loadCurrentCustomer(myInstance)` in `website/src/app/services/auth.ts` (no-op unless `isAuthedAs(..., ["CUSTOMER"])`) calls `LoadCurrentCustomer`; invoked from `boot.server` in `website/src/app/services/index.ts` after start + permissions — **before** `applyRouterMiddleware` so the org setup gate can read `me.organization`.
- Org setup gate (incomplete org → force `CustomerOrganization`): `flow-customer-organization.md` §3.1; `flow-auth.md` §2.1.
- Socket registry key must be `OnCustomerEvent` (mirrors backend notify event name) in `website/src/resources/configs/socket/events.ts`.

## 7) Home page — command map

Authed customer landing via `getMyHomeIdentify` → `"CustomerHome"` (`/customer`, `CUSTOMER_MAIN`, no breadcrumb / no `CustomerSubHeader`).

### Entry

| Path | Role |
|---|---|
| `website/src/app/ui/pages/customer/CustomerHome.tsx` | Thin `MyPage` → `CustomerHomeScreen` |
| `website/src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | Composition root |
| `website/src/app/ui/components/customer/hooks/useCustomerHome.ts` | Probes + derived model |

### Job

Org-admin **follow-up / command map**: what to do and what to watch. Customer is the workspace manager (not a meeting participant). `STARTED` = monitor via details — **no** LiveKit join CTA.

### Zones (hide when empty / N/A)

1. **Command hero** — brand tint surface, brackets, eyebrow marker, org name, subscription label, focus status card, featured CTA (navy-led, orange accent rail — not solid inverted orange), focus SVG quorum/readiness ring (`HomeFocusVisual`, HeroDashboard DNA + live counts)
2. **Workspace lifecycle rail** — organization → members → channels → templates → meetings
3. **Needs attention** — max 5 actionable rows
4. **Focus readiness + agenda strip** — readiness only for focus `DRAFT`; agenda when focus has items
5. **Action status cards** — members / channels / templates / meetings counts (brand wells; warn = accent rail only)
6. **Others in same status** — up to remaining meetings in the focus status bucket (excludes focus)

### Data

Private `useShallowAdapter` ids (do **not** share directory/history adapters):

- `customer-home-members` / `channels` / `templates` — `total_count` probes
- `customer-home-meetings-draft|waiting|started` — status-filtered slices (UI cap 5)
- `customer-home-focus-meeting` — `meeting(id)` for readiness (same facts as details, no can* abilities)
- `customer-home-me-ability` — `me.canCreateMeeting`
- Org + subscription from existing `useMe()`

Focus priority: `STARTED` > `DRAFT` > `WAITING_TO_START`. Primary CTA: no members → add members; else no focus + can create → create; else continue/review/monitor by focus status.

### Landing motif reuse (not a marketing clone)

Reuse craft in `customer/home/*`: eyebrow zones (`HomeZoneShell`), lifecycle rail, featured accent CTA, attention rows, readiness stratum, hero brackets, optional `Reveal` from `components/home/Reveal.tsx`. **Do not** port Problem/Roles/Trust/Impact/Faq, ChaosIllustration, GovernanceGraph, sparklines, or fake demo dashboard data.

### i18n

`ui.pages.customer.home.*` in `ar.ts` / `en.ts`.

### Non-goals

- LiveKit join / live session UI
- Navigation to unshipped `CustomerSubscription`
- Fake KPI / Impact sparklines
- Approving from home (approve stays on details)

### Runtime and failure behavior

`useCustomerHome` performs only customer-scoped reads. It owns no requester write, transaction, approval, or live-session action:

- Each mount-private adapter loads with `reload: true`. The meeting slices must finish before a focus is selected; then `customer-home-focus-meeting` loads the full focused meeting.
- The hero may initially render the focused status slice while the full meeting loads. Draft readiness, the agenda strip, quorum values, and attention for an incomplete focus draft appear only after the full focused meeting is available.
- Totals use backend `total_count`; status slices are capped to five records for display. Focus priority is `STARTED`, then `DRAFT`, then `WAITING_TO_START`.
- `HomeFocusVisual` derives its quorum ring from the real `min_members_count` and participant count. It has no fabricated fallback count. The center percent label resolves `getColor(semanticColor.textPrimary)` so it stays readable on the scheme-flipped hub fill (`cardBackground`); navy/`primaryActionBackground` is wrong for that glyph (dark-on-dark in dark mode).
- If all meaningful probes fail before data is available, the page renders its retry state. If only some probes fail, the available zones remain visible and an inline retry reloads every home probe, including the current focus meeting.
- The home hook has no page-local socket subscription. It reflects its mount reads and explicit retry; meeting preparation, approval, notifications, and live-session interactions remain owned by their existing screens.

### Component ownership

| Path | Responsibility |
|---|---|
| `website/src/app/ui/pages/customer/CustomerHome.tsx` | Route entry only; mounts `CustomerHomeScreen`. |
| `website/src/app/ui/components/customer/hooks/useCustomerHome.ts` | Private read adapters, focus selection, totals, lifecycle state, CTA priority, attention, readiness, retry. |
| `website/src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | Translates the view model and composes zones, retry UI, cards, and navigation. |
| `website/src/app/ui/components/customer/home/CustomerHomeCommandHero.tsx` | Hero structure, decorative brackets, and shared eyebrow marker. |
| `website/src/app/ui/components/customer/home/CustomerHomeFeaturedCta.tsx` | Primary next-step card; delegates interaction to `FormActionButton`. |
| `website/src/app/ui/components/customer/home/CustomerHomeStatusCard.tsx` | Focus-meeting status summary and typed details navigation. Renders `MeetingMetaChips` (type/status/notify) — **not** a joined `metaLine`; screen passes `*Label` + `*Value` (same contract as directory cards — `flow-customer-meetings.md` §4.1–§4.2). |
| `website/src/app/ui/components/customer/home/HomeZoneShell.tsx` | Local zone heading, eyebrow marker, and optional branded surface. |
| `website/src/app/ui/components/customer/home/WorkspaceLifecycleRail.tsx` | Responsive organization-to-meetings lifecycle navigation. |
| `website/src/app/ui/components/customer/home/HomeAttentionList.tsx` | Capped attention rows with optional typed navigation. |
| `website/src/app/ui/components/customer/home/HomeReadinessStratum.tsx` | Draft approval prerequisite rows. |
| `website/src/app/ui/components/customer/home/HomeAgendaStrip.tsx` | Focus agenda preview with horizontal overflow. |
| `website/src/app/ui/components/customer/home/HomeActionStatusCards.tsx` | Workspace totals and directory navigation. |
| `website/src/app/ui/components/customer/home/HomeFocusVisual.tsx` | Decorative, data-backed quorum/readiness SVG; hidden from assistive technology because equivalent data is exposed in the status and readiness UI. Hub **percent** SVG `fill` uses `semanticColor.textPrimary` (scheme-flipped with `cardBackground`); do **not** use `primaryActionBackground` (navy stays dark in both schemes → invisible on dark card). Seat strokes may still use navy. |
| `website/src/app/ui/components/customer/meetings/CustomerMeetingCard.tsx` | Reused meeting card; renders `MeetingMetaChips` (type/status/notify pills). Home passes `*Label` + `*Value` discriminators (`notify_status` now on the list query); see `flow-customer-meetings.md` §4.1–§4.2. |
| `website/src/resources/translations/ar.ts` | Arabic `ui.pages.customer.home.*` copy. |
| `website/src/resources/translations/en.ts` | English mirror of the home-copy tree. |

## 7.1) Members page + sub-header breadcrumb

### Route + page

- `CustomerMembers` — `MyPage` at `/customer/members` (`CUSTOMER_MAIN`, `mustAuthedAs: ["CUSTOMER"]`) mounting `CustomerMembersScreen` (directory UI).
- Route `breadcrumb`: `{ parent: "CustomerHome", label: tr => tr.ui.pages.customer.members.title }` (W38).
- Drawer tile `CustomerMembers` enables automatically once the identify is registered.
- List/search/load-more contract: **`flow-customer-members.md`** (not duplicated here).

### Shell composition

- Fixed `CustomerSubHeader` (sibling of `CustomerHeader`, not inside the content `Box`) when `routes[identify]?.breadcrumb` is set.
- Content `paddingTop` includes `CUSTOMER_SUBHEADER_BAR` (`2.75rem`) on subpages (responsive header heights).
- `CustomerSubHeader` → `useBreadcrumbs({ rootIdentify: "CustomerHome", rootLabel, rootMark: <HomeMark className="bc-crumb-mark"/> })` → `Breadcrumb`.
- Root crumb label/mark come from the **caller root** args (not route `rootLabel`) unless a descendant supplies W35 `rootLabel` / `rootIcon` / `rootNavParams`.

### Product placement (not `ui/base/`)

| Artifact | Path |
|---|---|
| Fixed bar | `components/customer/CustomerSubHeader.tsx` |
| Trail UI | `components/Breadcrumb.tsx` |
| Walker hook | `components/useBreadcrumbs.ts` |
| Home mark | `components/HomeMark.tsx` |
| Contract types | `types/extends/global.ts` → `BreadcrumbMeta` on `PageRouteState` |

Do **not** put breadcrumb product hooks/helpers under `ui/base/`. `ui/base/hooks` stays framework infrastructure (`useRouter`, `useNav`, …).

### Breadcrumb visual contract (brand trail)

Not a classic chevron trail. Observable behavior:

| Element | Behavior |
|---|---|
| Separator | Accent orange dot (`accentActionBackground`), not chevron |
| Ancestors | Secondary ink; optional Feather `Icon` **or** `mark` ReactNode; hover → accent text color (`getColor` + `.bc-crumb-ink`); mark opacity dip via `.bc-crumb-mark` |
| Current | Primary bold text + **2px accent underline rail** under the crumb (header-rail echo) |
| Overflow | Horizontal scroll, hidden scrollbar, `scrollIntoView` last crumb |
| Nav | `onClick` → `nav.push` (not `Link`) |

### `HomeMark`

Utils-composed dual-tone mark (portal ink/white frame + accent floor rail):

| Consumer | Props |
|---|---|
| Breadcrumb root | default `tone="ink"`, `className="bc-crumb-mark"` |
| Drawer home tile | `tone="onPrimary"`, `size={1.45}` |

Governance: `.cursor/rules/website-customer-utils-composed-marks.mdc`.

## 8) Bottom bar — planned

`CustomerBottomBar` + `BottomIcons` and `Dims.bottomBarHeight` are **not** in the website tree yet. Prior contract target (three items: Home / Notifications / Settings) remains deferred; do not invent clearance tokens until that stage.

## 9) Theme + z-index (shipped usage)

| Token / value | Consumer |
|---|---|
| `Dims.headerHeight` / `Dims.mobileHeaderHeight` | `CustomerMainLayout` content padding; header height |
| `zIndex.header` | `CustomerHeader` |
| `zIndex.modals` | `CustomerDrawer` portal |
| `semanticDims.shell.drawerWidth` | Drawer panel |
| Accent orange | Header rail, `DrawerMenuIcon` rail, `HomeMark` floor rail, breadcrumb separators + current underline, drawer tile hover well |
| `CUSTOMER_SUBHEADER_BAR` (`2.75rem`) | Fixed sub-header height; layout content `paddingTop` offset |

## 10) i18n

`website/src/resources/translations/ar.ts` + `en.ts` → `ui.layouts.customerMainLayout`:

- `header.*` — menu, greeting, defaultName, home/notifications aria
- `footer.rights`
- `drawer.*` — title (**تنقل سريع** / **Quick navigation**), close, nine tile labels, prefs, logout; workspace subtitle from `me.organization.name` or `workspaceFallback` (**منصة اجتماع**)
- `subHeader.navHome` / `subHeader.breadcrumbAriaLabel` — breadcrumb root label + nav aria
- `ui.pages.customer.members.title` — route breadcrumb leaf label for `CustomerMembers` (full members copy set: `flow-customer-members.md` §9)

Shipped tile label contract (user-approved wording):

| Key | ar | en |
|---|---|---|
| `itemHome` | الرئيسية | Home |
| `itemMeetings` | الاجتماعات | Meetings |
| `itemMembers` | الأعضاء | Members |
| `itemOrganization` | المؤسسة | Organization |
| `itemMessageTemplates` | قوالب الرسائل | Message templates |
| `itemSubscription` | الاشتراك | Subscription |
| `itemMessageChannels` | قنوات الإرسال | Message channels |
| `itemSettings` | الإعدادات | Settings |
| `itemSupport` | الدعم الفني | Support |
| `itemHelp` | المساعدة | Help |

Do not use longer forms such as «معلومات المؤسسة» or «الخطط / الاشتراك» unless product reopens the copy.

## 11) Route-reactive constraints (W24)

- `CustomerDrawer` — **without** `withMemo`.
- `CustomerMainLayout`, `CustomerHeader`, `CustomerFooter`, `HeaderIconButton`, `IdentityAvatar`, `DrawerMenuIcon` — `withMemo`.
- `CustomerSubHeader` — **without** `withMemo`; `Breadcrumb` may keep `withMemo` (crumbs are props).
- Future `CustomerBottomBar` — without `withMemo` when added.

See `docs/invariants/website.md` W24 and `.cursor/rules/website-route-reactive-components.mdc`.

## 12) Traceability map (members + breadcrumb change set)

| Path | Status | Doc anchor |
|---|---|---|
| `website/src/app/ui/pages/customer/CustomerMembers.tsx` | page host | §7.1; directory body → `flow-customer-members.md` |
| `website/src/app/ui/components/customer/CustomerSubHeader.tsx` | added | §2, §7.1 |
| `website/src/app/ui/components/Breadcrumb.tsx` | added | §7.1 |
| `website/src/app/ui/components/useBreadcrumbs.ts` | added | §7.1 |
| `website/src/app/ui/components/HomeMark.tsx` | added | §5.3, §7.1 |
| `website/src/app/ui/layouts/CustomerMainLayout.tsx` | modified | §2 |
| `website/src/app/ui/components/customer/CustomerDrawer.tsx` | modified | §5.2 workspace chip; §5.3 (`HomeMark` + `mark` branch) |
| `website/src/resources/configs/routes.ts` | modified | §7.1; `route-registry-contract.md` |
| `website/src/types/extends/global.ts` | modified | §7.1 (`BreadcrumbMeta`) |
| `website/src/resources/translations/ar.ts` / `en.ts` | modified | §10 |
| `website/lib/tsconfig.tsbuildinfo` | generated | skip narrative |

Prior shell inventory (header/footer/drawer/`useMe`/…) remains in earlier go-doc passes; see §3–§6 and related docs.

### 12.1 Home focus chips + hero percent contrast (this go-doc slice)

Exhaustive inventory for aligning the home focus status card with `MeetingMetaChips` and fixing dark-mode contrast on the hero quorum percent.

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/home/CustomerHomeStatusCard.tsx` | modified | Dropped `metaLine`; renders `MeetingMetaChips`; `*Label`/`*Value` props | §7 Component ownership |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | modified | Focus card passes chip props (no `·` join / no `NotStarted` notify gate) | §7 |
| `src/app/ui/components/customer/home/HomeFocusVisual.tsx` | modified | Percent SVG `fill` → `textPrimary` (was `primaryActionBackground`) | §7 Runtime / ownership |
| `lib/tsconfig.tsbuildinfo` | modified (generated) | TS incremental build cache; no narrative | generated |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role | Doc |
|---|---|---|---|
| `docs/platforms/website/flow-customer-shell.md` | modified | Status card chips + hero percent contrast; this inventory | §7, §12.1 |
| `docs/platforms/website/flow-customer-meetings.md` | modified | §12.4 cross-slice inventory | meetings §12.4 |
| `.cursor/rules/website-meeting-meta-chips.mdc` | modified | Explicit home status-card consumer | meetings §4.2 |
| `.cursor/rules/website-semantic-color-token-discipline.mdc` | modified | SVG text on scheme-flipped surfaces | §7 / W43 |

## 13) Related

- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/website/flow-customer-members.md`
- `docs/platforms/website/shared-ui-and-shell.md`
- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/ssr-boot-and-startup.md`
- `docs/platforms/website/flow-auth.md`
- `docs/invariants/website.md` (W24, W29, W35, W38)
- `.cursor/rules/website-authed-drawer-subpage-governance.mdc`
- `.cursor/rules/website-breadcrumb-descendant-root-label.mdc`
- `.cursor/rules/website-breadcrumb-product-placement.mdc`
- `.cursor/rules/website-customer-drawer-nav-backend-alignment.mdc`
- `.cursor/rules/website-customer-utils-composed-marks.mdc`
- `.cursor/rules/website-route-reactive-components.mdc`
- `.cursor/rules/website-utils-style-prop-precedence.mdc`
- `.cursor/rules/website-meeting-meta-chips.mdc`
- `.cursor/rules/website-semantic-color-token-discipline.mdc`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
