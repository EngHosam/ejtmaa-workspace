# Website Component and Folder Structure

Customer portal contract (see `overview.md`).

## Purpose

Ownership map for `website/` so feature work stays in the correct layer: infrastructure, shared UI, layouts, pages, services, and types.

Actors: **visitor** and **customer**.

## 1) Top-level layout

| Path | Owns |
|---|---|
| `src/client/*`, `src/server/*` | Browser and SSR entrypoints |
| `src/resources/configs/*` | Routes, store, axios, theme, web-core |
| `src/resources/translations/*` | Localization source |
| `src/app/services/*` | Auth, router, boot, socket |
| `src/app/ui/base/*` | Framework infrastructure (`MyApp`, `MyPage`, hooks) |
| `src/app/ui/components/*` | Shared product UI |
| `src/app/ui/layouts/*` | Shell layouts |
| `src/app/ui/pages/*` | Route entry pages |
| `src/types/*` | Shared contracts and GQL mirrors |

### 1.1) Shipped vs planned

**Shipped layouts:** `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`), `CustomerMainLayout` (`CUSTOMER_MAIN`), `MeetingLayout` (`MEETING`).

**Shipped pages:** `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error`, `CustomerHome` (command map — `flow-customer-shell.md` §7), `CustomerMembers` (directory — `flow-customer-members.md`), `CustomerMemberForm`, `CustomerMeetings` / `CustomerMeetingForm` / `CustomerMeetingDetails` (`flow-customer-meetings.md`), `CustomerOrganization` (`flow-customer-organization.md`), `CustomerMessageChannels` / `CustomerMessageChannelForm` (`flow-customer-message-channels.md`).

**Shipped shared components:** `Header`, `Drawer`, `Footer`, `Logo`, `DrawerMenuIcon`, `HeaderIconButton`, `HomeMark`, `Breadcrumb`, `useBreadcrumbs`, `LandingHeader`, `LandingFooter`, `LandingMobileDrawer`, `ThemeModeSwitch`, `LanguageSwitch`, `Loadable`, `Toast`, `DataTable`, `ResultLane`, `CardSkeleton`, `LoadMoreButton`, `SearchField`, `SectionHeading`, `FilterOptionChip`, `FilterOptionChips`, `Wrong` (`Empty` / `LaneFailed`), auth (`AuthPageShell`, `AuthTextField`, `AuthNavLink`, `AuthSecondaryNavButton`), form (`FormTextField`, `FormActionButton`, `FormInputWrapper`, `FormChoiceField`, `FormEntityPickerField`, `FormDateTimeField`, …), modals (`FormModal` chrome, `EntityPickerModal`, `DateTimePickerModal`, `ConfirmModal`, `SelectableEntityCard`, `entity-picker/configs/*`), landing home sections (`home/*`), customer shell (`CustomerHeader`, `CustomerFooter`, `CustomerDrawer`, `CustomerSubHeader`, `IdentityAvatar`, `hooks/useMe`, `hooks/useCustomerMembers`, `hooks/useCustomerMeetings`, `hooks/useCustomerMessageChannels`, `members/*`, `meetings/*`, `modals/*` registered form modals, `message-channels/*`, `message-templates/*`).

**Planned (target contract, not yet in scaffold):** `CustomerBottomBar` / `BottomIcons`, remaining `pages/customer/*` workspace screens (subscription, settings, notifications, static info, support), Google social auth UI (`SelectableCard`, `Checkbox` on Register).

## 2) `ui/base/` -- infrastructure only

Framework-facing primitives: `MyApp`, `MyHtml`, `MyPage`, base hooks, form/adapter wrappers, `Utils.tsx`.

Business UI does not belong here.

## 3) `ui/components/` -- shared product UI

Shared project UI composed from `Utils` + semantic theme tokens.

Shipped groups:

| Group | Shipped examples |
|---|---|
| Feedback | `Loadable`, `Toast`, `Wrong` (`Empty`, `LaneFailed`) |
| List lane | `ResultLane`, `CardSkeleton`, `LoadMoreButton`, `SearchField`, `SectionHeading`, `FilterOptionChip` (landing text + orange underline), `FilterOptionChips`, `FilterCountChips` (label + count badge; optional color chrome; no embedded i18n) |
| Shell (generic) | `Header`, `Drawer`, `Footer`, `Logo`, `DrawerMenuIcon`, `HeaderIconButton` (shared by `CustomerHeader` + `MeetingHeader`) |
| Shell (visitor landing) | `LandingHeader`, `LandingFooter`, `LandingMobileDrawer` |
| Auth | `AuthPageShell`, `AuthTextField`, `AuthNavLink`, `AuthSecondaryNavButton` |
| Form | `FormTextField`, `FormActionButton`, `FormInputWrapper`, `FormChoiceField`, `FormEntityPickerField`, `FormDateTimeField` |
| Modals (shared visitor + customer) | `FormModal` (presentational chrome only), `EntityPickerModal`, `DateTimePickerModal`, `ConfirmModal`, `SelectableEntityCard`, `entity-picker/configs/*` |
| Home (visitor) | `home/Hero`, `home/Platform`, … (landing sections) |
| Customer home | `customer/home/*`, `customer/hooks/useCustomerHome` |
| Customer members | `customer/members/*`, `customer/hooks/useCustomerMembers` |
| Customer meetings | `customer/meetings/*` (screens, rows, cards, `MeetingMetaChips`, `MeetingNote` alert chrome, `MeetingParticipantGroup`, and the UI-only mirrors `meetingNotifyTemplateMode` / `meetingScheduleLead`), `customer/hooks/useCustomerMeetings` |
| Customer form modals | `customer/modals/*` (`MeetingBasicsModal`, `MeetingParticipantAddModal`, `MeetingSubjectModal` — registered; see `flow-form-foundation.md` §3.8b) |
| Live meeting (organization host) | `meeting/hooks/useMeetingLive` (`MeetingLiveProvider` + `useMeetingLive`), `meeting/hooks/useMeetingLiveMe`, `meeting/hooks/useMeetingAttendWindow` (clock; called only from session), `meeting/hooks/useMeetingLiveSession` (resolve + provider → `linking` / `can` / `actions` / `meeting` / `me` / `attendWindow`), `meeting/MeetingLinkingScreen` (FAILED uses shared `LaneFailed`) — `organization-host-routing.md` §5.1, §5.2, §5.3 |
| Meeting broadcast (LiveKit) | `meeting/hooks/useMeetingLiveKitToken` (join fetch union), `meeting/hooks/useMeetingLiveKitRoom` (`Room.connect`, peers, publish/playback, mute-all), `meeting/MeetingLiveBroadcast` (stage + attach components + camera/mic/sound controls + chair mute-all + non-chair raise-hand + tile queue/floor badges), `meeting/MeetingHandRaisedOffIcon` (local hand-with-slash `IconType`) — `flow-meeting-broadcast.md` |
| Meeting shell (organization host) | `meeting/MeetingHeader`, `meeting/MeetingHeaderMe`, `meeting/MeetingFooter`, `meeting/MeetingDrawerPanel`, `meeting/MeetingDrawerOverlay`, `meeting/MeetingLiveBroadcast`, `meeting/MeetingPageOverlay` (solid floating sheet while `STARTED`; `ph=page.padY`), `meeting/MeetingPrimaryButton`, `meeting/MeetingMetaChip`, `meeting/MeetingAttendanceCard` (own chrome; not HeaderMe), `meeting/MeetingAgendaCard`, `meeting/MeetingTalkQueueCard`, `meeting/hooks/useMeetingAttendance` (data VM; no navigation), `meeting/hooks/useMeetingAgenda`, `meeting/hooks/useMeetingTalkQueue` (queue rows + floor VM), `meeting/hooks/useOrganization`, `meeting/hooks/useMeetingPage` (`MeetingPageProvider`), `meeting/pages/*` (`MeetingInitPage` lobby reads session `attendWindow`; `MeetingLivePage` waiting/start only; `MeetingAttendancePage` chair log + page-owned bounce; `MeetingAgendaPage`; `MeetingTalkQueuePage` chair queue admin + page-owned bounce; remaining drawer ids via named pages + shared `MeetingPageStub` until product UI) — `organization-host-routing.md` §5.4, §5.5 (READY shell only; linking gate precedes it); IA: `.cursor/rules/website-meeting-shell.mdc` |

Planned groups (target contract):

| Group | Planned examples |
|---|---|
| Shell (visitor) | `LandingSubHeader`, `LandingDrawer` |
| Shell (customer) | Shipped: `CustomerHeader`, `CustomerFooter`, `CustomerDrawer`, `CustomerSubHeader`, `Breadcrumb`, `useBreadcrumbs`, `HomeMark`, helpers; planned: `CustomerBottomBar`, `BottomIcons` |
| Auth (social) | `SelectableCard`, `Checkbox` on Register |
| Form | `FormActionChip`, `FormAvatarField` |
| Static info | `HelpGuideScreen`, `AboutEjtmaaScreen`, `TermsConditionsScreen` |
| Home (visitor) | `home/*` landing sections for public `Home` route |
| Notifications | `customer/notifications/CustomerNotificationCard`, `NotificationRowSkeleton` |

Local `graphql.config.yml` is allowed when the subtree hosts GraphQL-aware code. Shipped helper configs point at `base.graphql` only; the target contract points at mirrored `base` + `customer` under `src/types/gql/`.

## 4) `ui/layouts/` -- shell layouts

| Layout | Use | Status |
|---|---|---|
| `BasicLayout` | Auth and error pages | shipped |
| `LandingLayout` | Public `Home` landing page | shipped |
| `MainLayout` | `UiMockup` and future authed subpages | shipped |
| `CustomerMainLayout` | Authed customer workspace (`CUSTOMER_MAIN`) | shipped |
| `MeetingLayout` | Organization-host `Meeting` route (`MEETING`) | shipped — `organization-host-routing.md` §5 |

`MyApp` resolves layout from route metadata.

## 5) `ui/pages/` -- route-owned pages

Thin route entry components bound to route identifiers.

Shipped public/auth: `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error`.

Organization host: `Meeting` (route + branded `MeetingLayout` shell; `Meeting.tsx` switches in-shell `MeetingPage` under `meeting/pages/`; while `STARTED`, persistent `MeetingLiveBroadcast` LiveKit stage + solid `MeetingPageOverlay` for non-`live` — `organization-host-routing.md` §5, §5.5; `flow-meeting-broadcast.md`).

Customer workspace: shipped `CustomerHome` command map, members directory + form, meetings directory + create form + details, organization settings, message channels directory, message templates directory. Planned: subscription, notifications, static info, support, bottom bar.

Pages orchestrate hooks, layouts, adapters, and shared UI. Extract reusable sections to `components/`.

## 6) `types/` -- contracts and GQL mirrors

| Path | Role |
|---|---|
| `types/requesters/requesters.website.ts` | Visitor and customer requester maps |
| `types/extends/global.ts` | `Layout`, `AuthedAs`, module augmentation |
| `types/gql/definitions/` | Mirrored SDL (`base`, `customer`) |
| `types/gql/gql-types/` | Generated types (`base`, `customer`) |
| `types/events.ts` | Socket event unions |

Website mirrors `base` + `customer` only.

## 7) Shared SSR layout with `cpanel/`

Both frontends share the same `@my-ssr/web-core` ownership model:

- one UI foundation (`Utils`, `theme.ts`),
- one infrastructure layer (`ui/base/`),
- one route layer (`ui/pages/`),
- one adapter layer,
- one form/requester layer,
- one translation source.

## 8) Quick decision table

| If the code is... | Place it here |
|---|---|
| SSR bootstrap | `src/client/` or `src/server/` |
| Route config | `src/resources/configs/routes.ts` |
| Auth/router/boot | `src/app/services/` |
| Framework plumbing | `src/app/ui/base/` |
| Reusable product UI | `src/app/ui/components/` |
| Page shell | `src/app/ui/layouts/` |
| Route screen | `src/app/ui/pages/` |
| Shared type or GQL mirror | `src/types/` |

## Related

- `docs/platforms/website/overview.md`
- `docs/platforms/website/shared-ui-and-shell.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/invariants/website.md` (W3, W5, W6)
