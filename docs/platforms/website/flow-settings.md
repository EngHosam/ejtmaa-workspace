# Website Customer Settings Flow

Customer portal contract (see `overview.md`).

## Scope

Drawer-opened settings for authenticated customers:
- Profile image update
- Password change (optional fields)

## Routes

| Identify | Path | Layout | Auth | Breadcrumb |
|---|---|---|---|---|
| `CustomerSettings` | `/customer/settings` | `CUSTOMER_MAIN` | `["CUSTOMER"]` | `parent: CustomerHome` |

## Form contract

- `useShallowForm` over `Forms.CUSTOMER_SETTINGS`
- Endpoint: `API.FORMS.CUSTOMER.R("customer")`
- Submit sub: `updateSettings`
- Fields: `avatar_file`, `current_password`, `new_password`
- No `readSettings` on mount — avatar preview from `useMe()`

## Success flow

- Main message auto-shown by `ResMainMessageMiddleware`
- After success: `useMe().update()` + `nav.back()`

## Backend

`CustomerRequester.updateSettings` on website platform. Password change invalidates other sessions when both password fields provided.

## Related

- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/backend/contracts/account-settings-requesters.md`
