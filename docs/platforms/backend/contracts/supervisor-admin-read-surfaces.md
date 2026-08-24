# Supervisor Admin Read Surfaces

## Scope

Supervisor-facing backend read surfaces that power cpanel administration:

- `me` — current supervisor profile (`canDeletePlan`)
- `home` — extras-only dashboard payload (`HomeBridge`)
- `notifications` — paginated notifications (unused by cpanel UI)
- `customers` / `customer(id)` / `customerStats`
- `organizations` / `organization(id)` (unused directory)
- `plans` / `plan(id)` / `planStats`
- `subscriptions` / `subscription(id)` / `subscriptionStats`
- `meetings` / `meeting(id)` / `meetingStats` (+ nested `_Member` chairperson)

Customer/stats detail: `supervisor-customers-and-stats.md`.  
Catalog / meetings / home: `supervisor-catalog-and-home.md`.  
Organization detail: `organization-domain.md`.

## GraphQL ownership

| Surface | Bridge | Schema |
|---|---|---|
| Home extras | `HomeBridge` | `SupervisorSchema` |
| Customer list/detail | `CustomerBridge` | `SupervisorSchema` |
| Customer stats | `CustomerStatsBridge` | `SupervisorSchema` |
| Organization list/detail | `OrganizationBridge` | `SupervisorSchema` |
| Plan list/detail | `PlanBridge` | `SupervisorSchema` |
| Plan stats | `PlanStatsBridge` | `SupervisorSchema` |
| Subscription list/detail | `SubscriptionBridge` | `SupervisorSchema` |
| Subscription stats | `SubscriptionStatsBridge` | `SupervisorSchema` |
| Meeting list/detail | `MeetingBridge` | `SupervisorSchema` |
| Meeting stats | `MeetingStatsBridge` | `SupervisorSchema` |
| Nested chairperson | `MemberBridge` | `SupervisorSchema` |
| Supervisor me | `MeBridge` | `SupervisorSchema` |
| Notifications | `NotificationBridge` | `SupervisorSchema` |

SDL: `backend/src/app/gql/definitions/supervisor.graphql`

## Frontend mirror boundary

- Customer GQL mirrors live in `website/` (`base` + `customer`) — active
- `cpanel/` consumes supervisor GQL for `me`, `home`, customers, meetings, plans, subscriptions — active; mirrors under `cpanel/src/types/gql/**`

When `cpanel/` returns, sync by command copy from backend SDL/types per `.cursor/rules/gql-schemas-bridges-general.mdc`.

## Related

- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/contracts/supervisor-catalog-and-home.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/cpanel/supervisor-admin-modules.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
