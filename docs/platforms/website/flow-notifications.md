# Website Customer Notifications Flow

Customer portal contract (see `overview.md`).

## Scope

Drawer/header-launched notifications list for authenticated customers:
- Route: `/customer/notifications`
- Read via customer GQL `notifications` query
- Write: `Delete all` via `NotificationRequester`
- Breadcrumb sub-header from route metadata

## Routes

| Identify | Path | Layout | Auth | Breadcrumb |
|---|---|---|---|---|
| `CustomerNotifications` | `/customer/notifications` | `CUSTOMER_MAIN` | `["CUSTOMER"]` | `parent: CustomerHome` |

## Data

- Adapter: `useShallowAdapter` with `enterMode: "FORCE_RELOAD_ON_MOUNT"`
- GQL: `notifications` list via `DATA_ADAPTERS.GQL`
- Form: `FORMS.CUSTOMER_NOTIFICATION` for delete-all action

## UI

- `CustomerNotificationsScreen` with `SectionHeading`, `FormActionChip` (delete all), notification cards
- `pt={"2rem"}` under layout offset (not `pv`)
- Pagination via `LoadMoreButton`

## Socket

`OnCustomerEvent` with `type: "UPDATED"` may trigger notification refresh.

## Related

- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/backend/contracts/socket-event-mirroring.md`
- `.cursor/rules/website-list-adapter-enter-mode.mdc`
