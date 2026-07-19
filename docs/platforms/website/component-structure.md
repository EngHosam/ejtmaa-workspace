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

**Shipped layouts:** `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`), `CustomerMainLayout` (`CUSTOMER_MAIN`).

**Shipped pages:** `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error`, `CustomerHome` (empty `Main`), `CustomerMembers` (directory — `flow-customer-members.md`), `CustomerMemberForm`, `CustomerMeetings` / `CustomerMeetingForm` / `CustomerMeetingDetails` (`flow-customer-meetings.md`), `CustomerOrganization` (`flow-customer-organization.md`), `CustomerMessageChannels` / `CustomerMessageChannelForm` (`flow-customer-message-channels.md`).

**Shipped shared components:** `Header`, `Drawer`, `Footer`, `Logo`, `DrawerMenuIcon`, `HomeMark`, `Breadcrumb`, `useBreadcrumbs`, `LandingHeader`, `LandingFooter`, `LandingMobileDrawer`, `ThemeModeSwitch`, `LanguageSwitch`, `Loadable`, `Toast`, `DataTable`, `ResultLane`, `CardSkeleton`, `LoadMoreButton`, `SearchField`, `SectionHeading`, `FilterOptionChip`, `FilterOptionChips`, `Wrong` (`Empty` / `LaneFailed`), auth (`AuthPageShell`, `AuthTextField`, `AuthNavLink`, `AuthSecondaryNavButton`), form (`FormTextField`, `FormActionButton`, `FormInputWrapper`, `FormChoiceField`, `FormEntityPickerField`, `FormDateTimeField`, …), modals (`EntityPickerModal`, `DateTimePickerModal`, `ConfirmModal`, `SelectableEntityCard`, `entity-picker/configs/*`), landing home sections (`home/*`), customer shell (`CustomerHeader`, `CustomerFooter`, `CustomerDrawer`, `CustomerSubHeader`, `HeaderIconButton`, `IdentityAvatar`, `hooks/useMe`, `hooks/useCustomerMembers`, `hooks/useCustomerMeetings`, `hooks/useCustomerMessageChannels`, `members/*`, `meetings/*`, `message-channels/*`).

**Planned (target contract, not yet in scaffold):** `CustomerBottomBar` / `BottomIcons`, remaining `pages/customer/*` workspace screens (templates, subscription, settings, notifications, static info, support), Google social auth UI (`SelectableCard`, `Checkbox` on Register).

## 2) `ui/base/` -- infrastructure only

Framework-facing primitives: `MyApp`, `MyHtml`, `MyPage`, base hooks, form/adapter wrappers, `Utils.tsx`.

Business UI does not belong here.

## 3) `ui/components/` -- shared product UI

Shared project UI composed from `Utils` + semantic theme tokens.

Shipped groups:

| Group | Shipped examples |
|---|---|
| Feedback | `Loadable`, `Toast`, `Wrong` (`Empty`, `LaneFailed`) |
| List lane | `ResultLane`, `CardSkeleton`, `LoadMoreButton`, `SearchField`, `SectionHeading`, `FilterOptionChip` (landing text + orange underline), `FilterOptionChips` |
| Shell (generic) | `Header`, `Drawer`, `Footer`, `Logo` |
| Shell (visitor landing) | `LandingHeader`, `LandingFooter`, `LandingMobileDrawer` |
| Auth | `AuthPageShell`, `AuthTextField`, `AuthNavLink`, `AuthSecondaryNavButton` |
| Form | `FormTextField`, `FormActionButton`, `FormInputWrapper`, `FormChoiceField`, `FormEntityPickerField`, `FormDateTimeField` |
| Modals | `EntityPickerModal`, `DateTimePickerModal`, `ConfirmModal`, `SelectableEntityCard` |
| Modals | `EntityPickerModal` (`ENTITY_PICKER`), `DateTimePickerModal` (`DATETIME_PICKER`), `SelectableEntityCard`, `entity-picker/configs/*` |
| Home (visitor) | `home/Hero`, `home/Platform`, … (landing sections) |
| Customer members | `customer/members/*`, `customer/hooks/useCustomerMembers` |
| Customer meetings | `customer/meetings/*`, `customer/hooks/useCustomerMeetings` |

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

`MyApp` resolves layout from route metadata.

## 5) `ui/pages/` -- route-owned pages

Thin route entry components bound to route identifiers.

Shipped public/auth: `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error`.

Customer workspace: shipped `CustomerHome`, members directory + form, meetings directory + create form + empty details, organization settings, message channels directory. Planned: templates, subscription, notifications, static info, support, bottom bar.

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
