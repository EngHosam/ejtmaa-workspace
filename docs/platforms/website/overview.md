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

- Requesters: `auth` (visitor), `customer`, `notification`
- GQL mirrors: `base` + `customer`
- Socket namespace: `/customer`
- Events: `OnCustomerEvent`

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

### 6.1) Path groups

Customer routes register through `customerRouter` in `routes.ts`.
Full path table, section blocks, and redirect rules: `route-registry-contract.md`.

Public routes: `Login`, `Home`, `Register`, `ResetPassword`, `UiMockup`.
Customer workspace routes live under `/customer/*`.
`Error` stays last in the registry.

## Documentation stance

Platform docs under `docs/platforms/website/` describe the customer portal contract: routes, GQL mirrors, requesters, flows, and UI ownership for visitor + customer actors on `/website`.

## Related

- `docs/platforms/website/README.md` — index
- `docs/platforms/website/repository-inventory.md` — file inventory
- `docs/invariants/website.md` — invariants
- `.cursor/rules/website-platform-governance.mdc` — governance rule
