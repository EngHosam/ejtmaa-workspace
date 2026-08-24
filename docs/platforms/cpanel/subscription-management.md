# CPanel Subscription Management (اشتراك)

Read-only supervisor **subscription** directory. Arabic UI: **اشتراك**. No Forms key.

Backend: `docs/platforms/backend/contracts/supervisor-catalog-and-home.md` §4.

## 1) Routes

| Identify | Path |
|---|---|
| `SupervisorSubscriptions` | `/supervisor/subscriptions` |
| `SupervisorSubscription` | `/supervisor/subscriptions/:id` |

Static list before `:id`. Nested `MPagesRoutes` params. Drawer: `FiCreditCard`, `alwaysAvailable`.

## 2) Adapters

List: `"supervisor-subscriptions"`, `listable: "subscriptions"`, history search key (customer name/email via backend). No status filter in the UI. `subscriptionStats` (`total_count`, `active_count`) on `Stats`.

Detail: no `adapterIdentify`, `section.subscription`. `PageStateLane` overlay flags on the hook. Contract: `docs/platforms/cpanel/supervisor-state-lanes.md`. `StatusChip` for subscription status.

Nested plan + customer on the card/detail as selected in the GQL strings.

## 3) Traceability

| Path | Role |
|---|---|
| `cpanel/.../subscriptions/*` | List/detail UI |
| `cpanel/.../pages/supervisor/SupervisorSubscriptions.tsx` | Thin list |
| `cpanel/.../pages/supervisor/SupervisorSubscription.tsx` | Thin detail |
| `backend/.../SubscriptionBridge.ts` | GQL |
| `backend/.../SubscriptionStatsBridge.ts` | Stats extras |

## Related

- `docs/platforms/cpanel/supervisor-state-lanes.md`
- `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`
