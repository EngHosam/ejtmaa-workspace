# Supervisor Admin Read Surfaces

## Scope

Supervisor-facing backend read surfaces that power cpanel administration:

- `me` — current supervisor profile
- `notifications` — paginated notifications
- `customers` / `customer(id)` — customer list and detail
- `customerStats` — customer aggregate KPIs

Detail: `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`.

## GraphQL ownership

| Surface | Bridge | Schema |
|---|---|---|
| Customer list/detail | `CustomerBridge` | `SupervisorSchema` |
| Customer stats | `CustomerStatsBridge` | `SupervisorSchema` |
| Supervisor me | `MeBridge` | `SupervisorSchema` |
| Notifications | `NotificationBridge` | `SupervisorSchema` |

SDL: `backend/src/app/gql/definitions/supervisor.graphql`

## Frontend mirror boundary

- `cpanel/` mirrors `base` + `supervisor`
- Customer GQL mirrors live in `website/` (`base` + `customer`)

Sync mirrors by command copy from backend SDL/types per `.cursor/rules/gql-schemas-bridges-general.mdc`.

## Related

- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/cpanel/supervisor-admin-modules.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
