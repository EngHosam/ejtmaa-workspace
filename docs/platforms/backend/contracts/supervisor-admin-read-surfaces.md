# Supervisor Admin Read Surfaces

## Scope

Supervisor-facing backend read surfaces that power cpanel administration:

- `me` — current supervisor profile
- `notifications` — paginated notifications
- `customers` / `customer(id)` — customer list and detail
- `customerStats` — customer aggregate KPIs
- `organizations` / `organization(id)` — organization list and detail
- nested `_Customer.organization`

Customer/stats detail: `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`.  
Organization detail: `docs/platforms/backend/contracts/organization-domain.md`.

## GraphQL ownership

| Surface | Bridge | Schema |
|---|---|---|
| Customer list/detail | `CustomerBridge` | `SupervisorSchema` |
| Customer stats | `CustomerStatsBridge` | `SupervisorSchema` |
| Organization list/detail | `OrganizationBridge` | `SupervisorSchema` |
| Supervisor me | `MeBridge` | `SupervisorSchema` |
| Notifications | `NotificationBridge` | `SupervisorSchema` |

SDL: `backend/src/app/gql/definitions/supervisor.graphql`

## Frontend mirror boundary

- Customer GQL mirrors live in `website/` (`base` + `customer`) — active
- `cpanel/` would mirror `base` + `supervisor` — **deferred** while the `cpanel/` checkout is temporarily absent; do not sync supervisor GQL mirrors now

When `cpanel/` returns, sync by command copy from backend SDL/types per `.cursor/rules/gql-schemas-bridges-general.mdc`.

## Related

- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/cpanel/supervisor-admin-modules.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
