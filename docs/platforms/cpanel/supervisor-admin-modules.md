# Cpanel Supervisor Admin Modules

## Scope

Supervisor-facing modules in `cpanel/` for Ejtmaa. Backend supports:
- Customer list and single-customer read
- Customer stats aggregate
- Notifications
- Account settings (supervisor password change)

## Route catalog

| Identify | Path | Role |
|---|---|---|
| `Home` | `/` | Supervisor home; `HomeStatCard` reads `customerStats.total_count` |
| `Customers` | `/customers` | Customer list via supervisor GQL |
| `Customer` | `/customer/:formType(show)/:id` | Single customer read and update |
| `AccountSettings` | `/account-settings` | Supervisor password change |

Platform routes (shell and auth):

| Identify | Path | Role |
|---|---|---|
| `Login` | `/login` | Supervisor auth |
| `UiMockup` | `/ui-mockup` | Shell review route |
| `Error` | `/:error(404\|500\|403)` | Explicit fallback page |

## Backend coupling

Supervisor GQL (`supervisor.graphql`):
- `customers` — paginated list
- `customer(id)` — single record
- `customerStats` — aggregate stats
- `organizations` / `organization(id)` + nested `_Customer.organization` — on backend supervisor SDL; cpanel mirror/UI deferred while platform checkout is absent
- `me`, `notifications`

Requesters on cpanel platform:
- `auth`, `supervisor`, `customer`, `website_settings`, `platform_settings`

## Shared patterns

- List routes: `useShallowAdapter` + `useWithHistoryState` for filter persistence
- Form routes: `useShallowForm` with `mapState` exposing `isLoading` / `saving`
- Reads via `DATA_ADAPTERS.GQL`; writes via `FORMS.SUPERVISOR.R`

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
