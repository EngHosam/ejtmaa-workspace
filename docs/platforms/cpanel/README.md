# CPanel Platform Documentation

Authoritative notes for the Ejtmaa supervisor control panel under `cpanel/`.

## Documents

| Document | Description |
|---|---|
| [overview.md](./overview.md) | Supervisor SSR foundation, boot, implemented routes, backend coupling |
| [supervisor-admin-modules.md](./supervisor-admin-modules.md) | Implemented routes vs deferred backend-ready modules |
| [repository-inventory.md](./repository-inventory.md) | Current repo file inventory |
| [component-structure.md](./component-structure.md) | Folder and layer ownership |
| [ui-foundation.md](./ui-foundation.md) | Utils + theme.ts UI contract |
| [data-flow-and-gql.md](./data-flow-and-gql.md) | Adapters, requesters, GQL |
| [graphql-mirror-and-tooling.md](./graphql-mirror-and-tooling.md) | GQL mirror sync |
| [main-layout-responsive-shell.md](./main-layout-responsive-shell.md) | `MainLayout` responsive shell |
| [login-runtime-and-feedback.md](./login-runtime-and-feedback.md) | Supervisor login flow |
| [error-route-and-guard.md](./error-route-and-guard.md) | `Error` route and guard bypass |
| [shared-shell-and-ui-mockup.md](./shared-shell-and-ui-mockup.md) | Shared shell and UiMockup |
| [customer-management.md](./customer-management.md) | Deferred customer module (backend contract only) |
| [../../invariants/cpanel.md](../../invariants/cpanel.md) | Cpanel invariants |

## Implementation baseline (bootstrap)

Checked-in `cpanel/` is a sibling of `website/`, copied from that project and stripped of Website product surface.

- Runtime: SSR web app with `web-core` configuration ownership.
- Implemented routes: `Login`, empty `Home`, `UiMockup`, `Error`.
- Layouts: `BASIC` (auth/error), `MAIN` (authed shell).
- Actor: `SUPERVISOR` after login; visitor scope for `auth.supervisorLogin`.
- GQL mirrors: `base` + `supervisor` under `src/types/gql/`.
- Reads: `GET /cpanel/data_adapters/gql`. Writes: supervisor requester forms (foundation).
- Localization: Arabic-only wiring (`locales: ["ar"]`).
- Package: `ejtmaa-cpanel`. Dev port: `3095`.
- Quality scripts: `yarn type-check`, `yarn package`, `yarn start`. There is no lint or test script.

Do not treat deferred supervisor modules (customers, account settings, home KPIs) as implemented.

## Related

- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `.cursor/rules/cpanel-platform-governance.mdc`
