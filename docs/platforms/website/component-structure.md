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

## 2) `ui/base/` -- infrastructure only

Framework-facing primitives: `MyApp`, `MyHtml`, `MyPage`, base hooks, form/adapter wrappers, `Utils.tsx`.

Business UI does not belong here.

## 3) `ui/components/` -- shared product UI

Shared project UI composed from `Utils` + semantic theme tokens.

Groups:

| Group | Examples |
|---|---|
| Feedback | `Loadable`, `GlobalLoadable`, `Toast` |
| Shell (visitor) | `LandingHeader`, `LandingSubHeader`, `LandingFooter`, `LandingDrawer` |
| Shell (customer) | `CustomerHeader`, `CustomerFooter`, `CustomerDrawer`, `CustomerBottomBar`, `BottomIcons` |
| Auth | `AuthPageShell`, `AuthTextField`, `FormActionButton` |
| Form | `FormTextField`, `FormActionChip`, `FormInputWrapper` |
| Static info | `HelpGuideScreen`, `AboutEjtmaaScreen`, `TermsConditionsScreen` |
| Home (visitor) | `home/*` landing sections for public `Home` route |
| Notifications | `customer/notifications/CustomerNotificationCard`, `NotificationRowSkeleton` |

Local `graphql.config.yml` is allowed when the subtree hosts GraphQL-aware code and points at mirrored `base` + `customer` schemas under `src/types/gql/`.

## 4) `ui/layouts/` -- shell layouts

| Layout | Use |
|---|---|
| `BasicLayout` | Auth and error pages |
| `LandingLayout` | Visitor landing and public marketing subpages |
| `CustomerMainLayout` | Authed customer workspace (`CUSTOMER_MAIN`) |
| `MainLayout` | Review and shell preview routes (`UiMockup`) |

`MyApp` resolves layout from route metadata.

## 5) `ui/pages/` -- route-owned pages

Thin route entry components bound to route identifiers.

Public/auth: `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error`.

Customer workspace: `pages/customer/*` (`CustomerHome`, settings, notifications, static info, support).

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
