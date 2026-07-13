# Account Settings Requesters (Ejtmaa)

**Primary sources:**
- `backend/src/app/orchestrator/requesters/CustomerRequester.ts`
- `backend/requesters.website.ts`
- `website/src/types/requesters/requesters.website.ts`
- `website/src/app/ui/components/customer/settings/CustomerSettingsScreen.tsx`
- `website/src/app/ui/components/form/FormAvatarField.tsx`

## Goal

Expose authenticated customer account-settings requesters on the `/website` mount without inventing a parallel persistence path outside requester ownership.

Settings surface:

- profile image update
- password update

Consumer: website customer settings route `/customer/settings` — see `docs/platforms/website/flow-settings.md`.

No cpanel/supervisor settings surface uses these subs.

## Registration

- `@requester("customer")`
  - `@sub(["website"], ["customer"]) readSettings`
  - `@sub(["website"], ["customer"]) updateSettings`

Requester map parity:

- `backend/requesters.website.ts`
- `website/src/types/requesters/requesters.website.ts`

## Read contract (`readSettings`)

Validated owner context: `facts: isByCustomerFacts(joi)`

Returned form `values`:

- `name`
- `email`
- `mobile`
- `avatar_file` (`string | null` uploaded filename, not a `FileObject`)

## Update contract (`updateSettings`)

Optional inputs:

- `avatar_file: string | null`
- `current_password?: string`
- `new_password?: string`

Validation: `isByCustomerFacts(joi, true)`

Runtime facts:

- `avatar_file` is the upload key from `multipart_upload`, not a local file payload
- avatar update is independent from password change
- password update runs only when both password fields are present

### Avatar persistence

When `avatar_file !== undefined`, the requester updates `customers.avatar_file` inside the requester transaction. Upload happens in the website form field layer; the requester persists only the stable filename.

### Password persistence

When both password fields are present:

1. compare `current_password` against the authenticated user
2. reject mismatch with `WRONG_CURRENT_PASSWORD`
3. update `users.password`
4. destroy all other tokens for the same user, except the current token when present

## Event side effects

After commit: `notify<OnCustomerEventRefs>("OnCustomerEvent", { type: "UPDATED", customer })`

## Website coupling

- `FormAvatarField` uploads immediately through `API.ACTIONS.MULTIPART_UPLOAD`
- the form stores the returned filename in `avatar_file`
- submit sends only `sub: "updateSettings"`
- success toasts are automatic via `ResMainMessageMiddleware` — see `.cursor/rules/website-form-success-toast-automatic.mdc`

## Failure handling

- invalid actor facts → Joi/authorization rejection
- wrong current password → `WRONG_CURRENT_PASSWORD`
- malformed `avatar_file` type → Joi validation rejection
- only one password field supplied → password unchanged
- `avatar_file` omitted → avatar untouched

## Related

- `docs/platforms/backend/contracts/http-and-requesters.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-form-foundation.md`
