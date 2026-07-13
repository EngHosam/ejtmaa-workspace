# CPanel Platform Documentation

Authoritative notes for the Ejtmaa supervisor control panel under `cpanel/`.

## Documents

| Document | Description |
|---|---|
| [overview.md](./overview.md) | Supervisor SSR foundation, boot, routes, backend coupling |
| [supervisor-admin-modules.md](./supervisor-admin-modules.md) | Supervisor module route catalog |
| [repository-inventory.md](./repository-inventory.md) | Repo file inventory |
| [component-structure.md](./component-structure.md) | Folder and layer ownership |
| [ui-foundation.md](./ui-foundation.md) | Utils + theme.ts contract |
| [data-flow-and-gql.md](./data-flow-and-gql.md) | Adapters, requesters, GQL |
| [graphql-mirror-and-tooling.md](./graphql-mirror-and-tooling.md) | GQL mirror sync |
| [main-layout-responsive-shell.md](./main-layout-responsive-shell.md) | `MainLayout` responsive shell |
| [login-runtime-and-feedback.md](./login-runtime-and-feedback.md) | Supervisor login flow |
| [error-route-and-guard.md](./error-route-and-guard.md) | `Error` route and guard bypass |
| [shared-shell-and-ui-mockup.md](./shared-shell-and-ui-mockup.md) | Shared shell and drawer |
| [customer-management.md](./customer-management.md) | Customer list and detail module |
| [../../invariants/cpanel.md](../../invariants/cpanel.md) | Cpanel invariants |

## Implementation baseline (contract)

Target supervisor cpanel contract. The checked-in `cpanel/` repository implements these contract surfaces.

- Runtime: SSR web app with `web-core` configuration ownership.
- Routes: `Login`, `Home` (`HomeStatCard` + `customerStats.total_count`), `Customers`, `Customer`, `AccountSettings`, `UiMockup`, `Error`.
- Layouts: `BASIC` (auth/error), `MAIN` (authed shell).
- Actor: `SUPERVISOR` after login; visitor scope for `auth.supervisorLogin`.
- GQL mirrors: `base` + `supervisor` under `src/types/gql/`.
- Reads: `GET /cpanel/data_adapters/gql`. Writes: supervisor requester forms.
- Localization: Arabic-only with RTL.

## Related

- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `.cursor/rules/cpanel-platform-governance.mdc`
