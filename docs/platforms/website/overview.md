# Website Platform Overview (Ejtmaa)

## Role

`website/` is the **Customer** SSR frontend for Ejtmaa, served on backend mount `/website`.

Actors: **visitor** (unauthenticated) and **customer** (authenticated).

## Architecture

Same `@my-ssr/web-core` family as `cpanel/`:
- `@my-ssr/web-core` + `@typescript/sys-core`
- `src/client/*`, `src/server/*`, `src/resources/configs/web-core.ts`
- Folder ownership: `resources/`, `services/`, `ui/base/`, `ui/components/`, `ui/layouts/`, `ui/pages/`, `types/`

## UI foundation

- `ui/base/components/Utils.tsx` — layout/styling primitives
- `resources/configs/theme.ts` — brand source of truth (navy `#0B2057`, orange `#EC6901`)
- `resources/configs/utils.ts` — theme path helpers

See `docs/platforms/website/ui-foundation.md`.

## Backend coupling

- Requesters: `visitor.auth`, `customer.customer`, `customer.notification` (see `src/types/requesters/requesters.website.ts`)
- GQL mirrors: `base` + `customer` (`me`, `notifications`, `me.canDeleteNotifications`)
- Socket namespace: `customer`
- Events: `onCustomerEventDate` (payload `OnCustomerEventDate { type: "UPDATED" }`)
- Backend mount: `/website` (test `http://192.168.1.10:3206/website`, prod `https://backend.ejtmaa.live/website`)

## Flow documentation

| Flow | Document |
|---|---|
| Auth | [flow-auth.md](./flow-auth.md) |
| Form foundation | [flow-form-foundation.md](./flow-form-foundation.md) |
| Settings | [flow-settings.md](./flow-settings.md) |
| Notifications | [flow-notifications.md](./flow-notifications.md) |
| Static info pages | [flow-static-info-pages.md](./flow-static-info-pages.md) |
| Customer shell | [flow-customer-shell.md](./flow-customer-shell.md) |

## 6) Route registry summary

### 6.1) Shipped routes

`src/resources/configs/routes.ts` currently ships four identifies:

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Home` | `/` | `MAIN` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `Error` | `/:error(404|500|403)` | `BASIC` (last entry) |

`publicRoutes` = `["Login", "UiMockup", "Home"]`. `getMyHomeIdentify` returns `"Home"`.

Layouts shipped: `BasicLayout` (`BASIC`) and `MainLayout` (`MAIN`).

### 6.2) Planned (not shipped)

`customerRouter`, the `/customer/*` workspace, `Register`, `ResetPassword`, `CustomerHome`, `CUSTOMER_MAIN` layout, and the `customer`/`base`/`Error` section-block split are planned per `route-registry-contract.md` and `website-route-registry-governance.mdc`. They are not yet present in `routes.ts`.

## Documentation stance

Platform docs under `docs/platforms/website/` describe the customer portal contract: routes, GQL mirrors, requesters, flows, and UI ownership for visitor + customer actors on `/website`.

## Related

- `docs/platforms/website/README.md` — index
- `docs/platforms/website/repository-inventory.md` — file inventory
- `docs/invariants/website.md` — invariants
- `.cursor/rules/website-platform-governance.mdc` — governance rule
