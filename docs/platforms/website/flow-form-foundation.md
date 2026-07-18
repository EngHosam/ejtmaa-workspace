# Website Form Foundation (Customer)

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native requester form foundation for `website/`: the typed requester route map, shared `FORMS` identities, route-scoped create/edit form screens, shared form field surfaces, and the `useForm` / `useShallowForm` / `FormInput` hook contract — all on the website web-core form stack.

## 2) Web-native approach (governance)

- **Requester endpoint typing** — `website/src/types/requesters/requesters.website.ts` (shipped):
  - `visitor.auth`: `registerCustomer` | `login` | `resetPassword`,
  - `customer.customer`: `readSettings` | `updateSettings`,
  - `customer.member`: `read` | `create` | `update` | `delete`,
  - `customer.notification`: `deleteAll`,
  - `customer.subscription`: `subscribe`.
  - Planned subs not yet shipped: `visitor.auth` `loginSocial` | `registerCustomerSocial` (Google social).
  - Endpoint builders (shipped): `API.FORMS.R("auth")(sub)` (visitor), `API.FORMS.CUSTOMER.R(requester)(sub)`.
- **Shared `FORMS` registrations** — `website/src/resources/configs/store/forms.ts` stores the requester endpoint builder (not a preselected `sub`); callers pass the truthful `sub` at send time.
  - `Forms.FORM1` — empty inherit for mockups / local tools.
  - `Forms.CUSTOMER_MEMBER` — `API.FORMS.CUSTOMER.R("member")` (shipped member create/edit).
- **Form hooks (web-core)** — same ownership split as cpanel:
  - `useForm` reads an already-existing form entry,
  - `useShallowForm` injects a form reducer entry on mount (route/modal/scoped forms),
  - the screen owns layout, `sub` selection, duplicate-submit guards (`submittingRef`), read-on-edit hydration, and after-success behavior,
  - field hooks own value/errors; submit handlers send only the `sub`,
  - temporary private `useShallowForm` forms use `removeOnExit: true` by default; create drafts that must survive SPA nav use stable `formIdentify` + `removeOnExit: false`.
- **Read-on-edit**: edit routes pass only identity; edit mode hydrates via a `read` sub on entry. Show `Loadable` during injection + edit `read` in flight.
- **Multi-path create+update**: one route identify with `path: { create, update }` — `.cursor/rules/website-multi-path-form-routes.mdc`. Typed href builders live under `resources/configs/customer/formRoute.ts`.
- **Customer form screens**:
  - auth: `/login`, `/register`, `/reset-password` — `API.FORMS.R("auth")`; see `flow-auth.md`,
  - member form: `/customer/members/form` (+ `/:id`) — `Forms.CUSTOMER_MEMBER`; see `flow-customer-members.md` §5,
  - account settings: `/customer/settings` — planned (`Forms.CUSTOMER_SETTINGS` not registered yet); see `flow-settings.md`,
  - notification delete-all — see `flow-notifications.md`.
- **Shared form field surfaces** under `src/app/ui/components/form/`:
  - `FormTextField`, `FormActionButton`, `FormAvatarField`, `FormInputWrapper`, `FormProvider`.
  - Compose from `Utils` + semantic theme tokens.

## 3) Field surface contracts (shipped)

### 3.1 `FormTextField`

- Controlled via `useFormInput({ name })`.
- Native direction follows the page locale (**do not** set Utils `ltr` / `ta_l` on the input). Use logical `textAlign: "start"` so Arabic caret/typing works.
- Rule: `.cursor/rules/website-form-text-field-direction.mdc`.
- Product convention for customer workspace forms: **no placeholders** unless the product owner explicitly asks.

### 3.2 `FormInputWrapper`

- Title: `variant="inputTitle"`.
- Subtitle: `variant="caption"` + `semanticColor.textTertiary` (must read quieter than the title).
- Errors: `semanticColor.stateError`.

### 3.3 `FormAvatarField`

- Form field name: **`avatar_file`** (filename string after upload; empty string clears on backend update when sent).
- Immediate multipart: `uploadEntityMediaFile` → `API.ACTIONS.MULTIPART_UPLOAD` → `uploadedName`.
- URI helpers: `mediaStaticRoot` / `mediaImageUri` (`website/src/app/helpers/media.ts`).
- Preview priority: local object URL → form `avatar_file` → optional `currentAvatarUrl`.
- Visual: compact navy identity circle (`primaryActionBackground` / `FiUser` / cover image) + quiet accent text action — Ejtmaa identity language (same family as `IdentityAvatar`), not Masdaria camera-chip / stretched input-shell chrome.
- Rule: `.cursor/rules/website-form-avatar-field.mdc`.

### 3.4 Success toasts

Automatic via `ResMainMessageMiddleware` — do not re-toast in `afterSentSuccess`. Rule: `.cursor/rules/website-form-success-toast-automatic.mdc`. Canonical member form: `CustomerMemberFormScreen`.

## 4) Verification checklist

- Website `forms.ts` registers identities storing the requester builder (not a preselected `sub`).
- `requesters.website.ts` mirrors backend `requesters.website.ts` unions (W18).
- Route-scoped forms use `useShallowForm` + `inheritedFormIdentify`; submit sends only the `sub`.
- `loading = !exist || !!formLoading`; `Loadable` during injection + edit `read` where applicable.
- Shared field surfaces live under `src/app/ui/components/form/`.
- Form success toasts are automatic — no duplicate manual toasts in `afterSentSuccess`.
- Multi-path forms use one identify + `formRoute` href builders.
- `yarn type-check` passes in `website/`.

## 5) Traceability (member form slice)

| Path | Role |
|---|---|
| `website/src/resources/configs/store/forms.ts` | `CUSTOMER_MEMBER` |
| `website/src/resources/configs/customer/formRoute.ts` | `buildCustomerMemberFormHref` |
| `website/src/types/requesters/requesters.website.ts` | `customer.member` |
| `website/src/app/ui/components/form/FormAvatarField.tsx` | avatar field |
| `website/src/app/helpers/entityMedia.ts` / `media.ts` | upload + URI |
| `website/src/app/ui/components/form/FormTextField.tsx` | direction |
| `website/src/app/ui/components/form/FormInputWrapper.tsx` | subtitle color |
| `docs/platforms/website/flow-customer-members.md` | end-to-end member UI |
| `docs/platforms/backend/contracts/member-domain.md` §9 | requester contract |

## 6) Related

- `docs/platforms/website/flow-auth.md`
- `docs/platforms/website/flow-customer-members.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-notifications.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/invariants/website.md` (W11, W18)
- `.cursor/rules/website-multi-path-form-routes.mdc`
- `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`
- `.cursor/skills/website-customer-member-form/SKILL.md`
