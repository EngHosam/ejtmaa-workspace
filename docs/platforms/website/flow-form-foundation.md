# Website Form Foundation (Customer)

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native requester form foundation for `website/`: the typed requester route map, shared `FORMS` identities, route-scoped create/edit form screens, shared form field surfaces, and the `useForm` / `useShallowForm` / `FormInput` hook contract — all on the website web-core form stack.

## 2) Web-native approach (governance)

- **Requester endpoint typing** — `website/src/types/requesters/requesters.website.ts`:
  - `visitor.auth`: `registerCustomer` | `registerCustomerSocial` | `login` | `loginSocial` | `resetPassword`,
  - `customer.customer`: `readSettings` | `updateSettings`,
  - `customer.notification`: `deleteAll`.
  - Endpoint builders: `API.FORMS.VISITOR.R("auth")(sub)`, `API.FORMS.CUSTOMER.R(requester)(sub)`.
- **Shared `FORMS` registrations** — `Forms.ts` stores the requester endpoint builder (not a preselected `sub`); callers pass the truthful `sub` at send time.
- **Form hooks (web-core)** — same ownership split as cpanel:
  - `useForm` reads an already-existing form entry,
  - `useShallowForm` injects a form reducer entry on mount (route/modal/scoped forms),
  - the screen owns layout, `sub` selection, duplicate-submit guards, read-on-edit hydration, and after-success behavior,
  - `FormInput` owns field value/errors/title layout; submit handlers send only the `sub`,
  - temporary private `useShallowForm` forms use `removeOnExit: true` by default.
- **Read-on-edit**: edit routes pass only identity; edit mode hydrates via a `read` sub on entry. Show `Loadable` during injection + edit `read` in flight.
- **Customer form screens**:
  - auth: `/login`, `/register`, `/reset-password` — see `flow-auth.md`,
  - account settings: `/customer/settings` — `Forms.CUSTOMER_SETTINGS` → `updateSettings`; see `flow-settings.md`,
  - notification delete-all — see `flow-notifications.md`.
- **Shared form field surfaces**: `FormTextField`, `FormActionButton`, `FormAvatarField` (settings), `FormInputWrapper`, `FormProvider`. Compose from `Utils` + semantic theme tokens.

## 3) Verification checklist

- Website `Forms.ts` registers identities storing the requester builder (not a preselected `sub`).
- Route-scoped forms use `useShallowForm` + `inheritedFormIdentify`; submit sends only the `sub`.
- `loading = !exist || !!formLoading`; `Loadable` during injection + edit `read` where applicable.
- Shared field surfaces live under `src/app/ui/components/form/`.
- Form success toasts are automatic via `ResMainMessageMiddleware` — no duplicate manual toasts in `afterSentSuccess`.
- `yarn type-check` passes.

## 5) Related

- `docs/platforms/website/flow-auth.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-notifications.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/invariants/website.md` (W11, W18)
