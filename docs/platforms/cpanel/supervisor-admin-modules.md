# CPanel Supervisor Admin Modules

## Scope

Supervisor-facing modules for Ejtmaa CPanel.

**Implemented in the current frontend:** Login, occupancy `Home` (`/`), dashboard `SupervisorHome` (`/supervisor`), customers / meetings / subscriptions read directories, plan catalog list+form, supervisor settings, Error. Shell identity via GQL `me`.

**Not implemented in the frontend:** notifications UI, organizations directory, customer writes, Support page. Backend still exposes org list/detail GQL and unused notification roots.

## Implemented routes

| Identify | Path | Role |
|---|---|---|
| `Login` | `/login` | Supervisor auth (`sub: "supervisorLogin"`) |
| `Home` | `/` | Occupies `/`; empty page; not in `publicRoutes` |
| `SupervisorHome` | `/supervisor` | Dashboard (`home` extras + aliased meetings) |
| `SupervisorCustomers` | `/supervisor/customers` | ResultLane + `customerStats` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | Read-only customer + nested institution |
| `SupervisorMeetings` | `/supervisor/meetings` | ResultLane + `meetingStats` |
| `SupervisorMeeting` | `/supervisor/meetings/:id` | Read-only meeting |
| `SupervisorPlans` | `/supervisor/plans` | ResultLane + `planStats` |
| `SupervisorPlanForm` | `/supervisor/plans/form` (+ `/:id`) | Requester `plan` create/update/delete |
| `SupervisorSubscriptions` | `/supervisor/subscriptions` | ResultLane + `subscriptionStats` |
| `SupervisorSubscription` | `/supervisor/subscriptions/:id` | Read-only subscription |
| `SupervisorSettings` | `/supervisor/settings` | Requester `supervisor` read/update |
| `Error` | `/:error(404\|500\|403)` | Explicit fallback page |

Contracts: `customer-management.md`, `meeting-management.md`, `plan-management.md`, `subscription-management.md`, `supervisor-settings.md`, `supervisor-home.md`.

## Deferred

Do not invent org routes, customer Forms, or a Support identify until those tasks ship.

Supervisor GQL still includes organizations and notifications unused by cpanel UI.

Requesters on the cpanel platform:

- visitor `auth` (`supervisorLogin`)
- supervisor `customer`, `plan`, `platform_settings`, `supervisor`, `website_settings`

List/detail reads: `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`. Home: `.cursor/skills/cpanel-supervisor-home/SKILL.md`. Writes: `useShallowForm` + `API.FORMS.SUPERVISOR.R`.

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/supervisor-catalog-and-home.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
