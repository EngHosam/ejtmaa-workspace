# CPanel Supervisor Admin Modules

## Scope

Supervisor-facing modules for Ejtmaa CPanel.

**Implemented in the current frontend:** Login, occupancy `Home` (`/`), empty `SupervisorHome` (`/supervisor`), Error. Shell identity via GQL `me`.

**Not implemented in the frontend.** Backend already exposes supervisor customer reads/stats and related requesters. Those remain backend contracts until a later CPanel product task ships UI for them. Do not invent dashboard widgets, customer CRUD screens, account-settings pages, or frontend paths for those modules in this bootstrap.

## Implemented routes

| Identify | Path | Role |
|---|---|---|
| `Login` | `/login` | Supervisor auth (`sub: "supervisorLogin"`) |
| `Home` | `/` | Occupies `/` so the mount root is not unmatched; empty page; not in `publicRoutes` |
| `SupervisorHome` | `/supervisor` | Authenticated `SUPERVISOR_MAIN` shell; page body is empty |
| `Error` | `/:error(404\|500\|403)` | Explicit fallback page |

## Deferred backend contracts (no CPanel UI, no frontend paths yet)

Do not treat the following as registered identifies or URL designs. A later product task must choose routes from this repository's Website DNA plus the backend `/cpanel` contract.

Supervisor GQL (`supervisor.graphql`) already includes customer list/detail/stats, organizations, `me` (shipped in chrome via `SUPERVISOR_ME`), and notifications.

Requesters on the cpanel platform:

- visitor `auth` (`supervisorLogin`)
- supervisor `customer`, `platform_settings`, `supervisor`, `website_settings`

When modules land:

- list reads: `useShallowAdapter` + history-backed search (Website list DNA)
- writes: `useShallowForm` + `API.FORMS.SUPERVISOR.R`
- reads via `DATA_ADAPTERS.SUPERVISOR_GQL`

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
