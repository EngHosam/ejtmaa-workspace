# Cpanel Repository Inventory

## Purpose

Inventory of project-owned paths under `cpanel/` for supervisor administration.

Platform docs describe the supervisor cpanel contract. Paths below are the contract inventory for the checked-in `cpanel/` repository.

## Runtime entry

| Path | Role |
|---|---|
| `src/client/index.ts` | Browser bootstrap |
| `src/server/index.ts` | SSR bootstrap |
| `src/resources/configs/web-core.ts` | Runtime configuration authority |
| `src/resources/configs/routes.ts` | Route registry |

## Services

| Path | Role |
|---|---|
| `src/app/services/index.ts` | Server/client boot phases |
| `src/app/services/auth.ts` | `SUPERVISOR` auth helpers |
| `src/app/services/router.ts` | Route guards and redirects |

## UI foundation

| Path | Role |
|---|---|
| `src/app/ui/base/components/Utils.tsx` | Layout/styling primitives |
| `src/resources/configs/theme.ts` | Brand tokens (when present in cpanel checkout) |
| `src/app/ui/base/core/MyApp.tsx` | Layout resolution |
| `src/app/ui/base/core/MyHtml.tsx` | Document shell |
| `src/app/ui/base/core/MyPage.tsx` | Route lifecycle |

## Shared shell

| Path | Role |
|---|---|
| `src/app/ui/layouts/BasicLayout.tsx` | Auth/error wrapper |
| `src/app/ui/layouts/MainLayout.tsx` | Authed shell (header, drawer, footer) |
| `src/app/ui/components/Header.tsx` | Shell header |
| `src/app/ui/components/Drawer.tsx` | Navigation drawer |
| `src/app/ui/components/Footer.tsx` | Shell footer |
| `src/app/ui/components/DataTable.tsx` | List table primitive |

## Route pages

| Path | Route |
|---|---|
| `src/app/ui/pages/Login.tsx` | `/login` |
| `src/app/ui/pages/Home.tsx` | `/` |
| `src/app/ui/pages/Customers.tsx` | `/customers` |
| `src/app/ui/pages/Customer.tsx` | `/customer/:formType(show)/:id` |
| `src/app/ui/pages/AccountSettings.tsx` | `/account-settings` |
| `src/app/ui/pages/UiMockup.tsx` | `/ui-mockup` |
| `src/app/ui/pages/Error.tsx` | `/:error(404|500|403)` |

## Home module

| Path | Role |
|---|---|
| `src/app/ui/components/home/HomeStatCard.tsx` | Home KPI from `customerStats.total_count` |
| `src/app/ui/components/home/hooks/useCustomerStats.ts` | `customerStats.graphql` shallow adapter |
| `src/app/ui/components/home/graphql/customerStats.graphql` | Supervisor stats operation |

## Customer module

| Path | Role |
|---|---|
| `src/app/ui/components/customer/CustomerStatsSection.tsx` | List header stat from `customerStats.total_count` |
| `src/app/ui/components/customer/graphql/customers.graphql` | Customer list operation |
| `src/app/ui/components/customer/graphql/customer.graphql` | Single customer operation |

## Types and GQL mirrors

| Path | Role |
|---|---|
| `src/types/requesters/requesters.cpanel.ts` | Supervisor requester map |
| `src/types/gql/definitions/base.graphql` | Mirrored base SDL |
| `src/types/gql/definitions/supervisor.graphql` | Mirrored supervisor SDL |
| `src/types/gql/gql-types/` | Generated types |
| `graphql.config.yml` | Local GQL tooling scope |

## Generated and bundled areas (not source of truth)

| Path | Role |
|---|---|
| `lib/` | TypeScript build output |
| `server/` | Generated SSR bundle |
| `eng-hosam/` | Bundled enterprise libraries |

## Related

- `docs/platforms/cpanel/supervisor-admin-modules.md`
- `docs/platforms/cpanel/component-structure.md`
