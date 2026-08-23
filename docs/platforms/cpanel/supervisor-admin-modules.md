# CPanel Supervisor Admin Modules

## Scope

Supervisor-facing modules for Ejtmaa CPanel.

**Implemented in the current frontend:** Login, occupancy `Home` (`/`), empty `SupervisorHome` (`/supervisor`), read-only customers list/detail, Error. Shell identity via GQL `me`.

**Not implemented in the frontend:** home KPI widgets, organizations module, customer writes, account settings. Backend still exposes org list/detail GQL and customer requesters for later tasks.

## Implemented routes

| Identify | Path | Role |
|---|---|---|
| `Login` | `/login` | Supervisor auth (`sub: "supervisorLogin"`) |
| `Home` | `/` | Occupies `/` so the mount root is not unmatched; empty page; not in `publicRoutes` |
| `SupervisorHome` | `/supervisor` | Authenticated `SUPERVISOR_MAIN` shell; page body is empty |
| `SupervisorCustomers` | `/supervisor/customers` | ResultLane directory + `Stats` from `customerStats` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | Read-only customer + nested institution |
| `Error` | `/:error(404\|500\|403)` | Explicit fallback page |

Customers module contract: `docs/platforms/cpanel/customer-management.md`.

## Deferred

Do not invent dashboard widgets, org routes, or customer Forms in this checkout.

Supervisor GQL still includes organizations and notifications unused by cpanel UI.

Requesters on the cpanel platform:

- visitor `auth` (`supervisorLogin`)
- supervisor `customer`, `platform_settings`, `supervisor`, `website_settings`

When further modules land, follow `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md` for list/detail reads (`useShallowAdapter` + `DATA_ADAPTERS.SUPERVISOR_GQL`). Writes: `useShallowForm` + `API.FORMS.SUPERVISOR.R`.

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
