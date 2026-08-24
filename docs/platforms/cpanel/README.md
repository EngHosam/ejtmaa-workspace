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
| [flow-supervisor-shell.md](./flow-supervisor-shell.md) | `SupervisorMainLayout` chrome |
| [route-registry-contract.md](./route-registry-contract.md) | `supervisorRouter`, `Home` occupancy, `publicRoutes` |
| [login-runtime-and-feedback.md](./login-runtime-and-feedback.md) | Supervisor login flow |
| [error-route-and-guard.md](./error-route-and-guard.md) | `Error` route and guard bypass |
| [customer-management.md](./customer-management.md) | Shipped read-only customers list/detail |
| [meeting-management.md](./meeting-management.md) | Read-only meetings directory |
| [plan-management.md](./plan-management.md) | Plan catalog (باقة) list + requester form |
| [subscription-management.md](./subscription-management.md) | Read-only subscriptions (اشتراك) |
| [supervisor-settings.md](./supervisor-settings.md) | Supervisor account settings form |
| [supervisor-home.md](./supervisor-home.md) | Authenticated home dashboard |
| [supervisor-state-lanes.md](./supervisor-state-lanes.md) | `ResultLane` / `PageStateLane` / `FormStateLane` overlay contract |
| [supervisor-shared-ui.md](./supervisor-shared-ui.md) | `StatusChip`, `FormMultiLangField`, `Stats` href/note |
| [supervisor-workspace-change-set.md](./supervisor-workspace-change-set.md) | Exhaustive path inventory for this wave |
| [../../invariants/cpanel.md](../../invariants/cpanel.md) | Cpanel invariants |

## Implementation baseline (bootstrap)

Checked-in `cpanel/` is a sibling of `website/`, copied from that project and stripped of Website product surface.

- Runtime: SSR web app with `web-core` configuration ownership.
- Implemented routes: `Login`, occupancy `Home` at `/`, dashboard `SupervisorHome`, customers / meetings / plans / subscriptions directories, plan form, settings, `Error`.
- Layouts: `BASIC` (auth/error), `SUPERVISOR_MAIN` (authed supervisor shell).
- Actor: `SUPERVISOR` after login; visitor scope for `auth.supervisorLogin`.
- GQL mirrors: `base` + `supervisor` under `src/types/gql/`.
- Reads: `GET /cpanel/data_adapters/supervisor/gql` (`SUPERVISOR_ME` for `me`; `SUPERVISOR_GQL` inherited by mount-private adapters). Writes: `plan` and `supervisor` requester forms. Customers / meetings / subscriptions UI is read-only.
- Localization: Arabic-only wiring (`locales: ["ar"]`).
- Package: `ejtmaa-cpanel`. Dev port: `3095`.
- Quality scripts: `yarn type-check`, `yarn package`, `yarn start`. There is no lint or test script.

Deferred: notifications UI, organizations directory, customer writes, Support page. Shipped modules: `customer-management.md`, `meeting-management.md`, `plan-management.md`, `subscription-management.md`, `supervisor-settings.md`, `supervisor-home.md`.

## Related

- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `docs/platforms/backend/contracts/supervisor-catalog-and-home.md`
- `.cursor/rules/cpanel-platform-governance.mdc`
- `.cursor/rules/cpanel-supervisor-home.mdc`
- `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`
