# Website Form Foundation (Customer)

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native requester form foundation for `website/`: the typed requester route map, shared `FORMS` identities, route-scoped create/edit form screens, shared form field surfaces, and the `useForm` / `useShallowForm` / `FormInput` hook contract — all on the website web-core form stack.

## 2) Web-native approach (governance)

- **Requester endpoint typing** — `website/src/types/requesters/requesters.website.ts` (shipped):
  - `visitor.auth`: `registerCustomer` | `login` | `resetPassword`,
  - `customer.customer`: `readSettings` | `updateSettings`,
  - `customer.member`: `read` | `create` | `update` | `delete`,
  - `customer.organization`: `read` | `upsert`,
  - `customer.notification`: `deleteAll`,
  - `customer.subscription`: `subscribe`.
  - Planned subs not yet shipped: `visitor.auth` `loginSocial` | `registerCustomerSocial` (Google social).
  - Endpoint builders (shipped): `API.FORMS.R("auth")(sub)` (visitor), `API.FORMS.CUSTOMER.R(requester)(sub)`.
- **Shared `FORMS` registrations** — `website/src/resources/configs/store/forms.ts` stores the requester endpoint builder (not a preselected `sub`); callers pass the truthful `sub` at send time.
  - `Forms.FORM1` — empty inherit for mockups / local tools.
  - `Forms.CUSTOMER_MEMBER` — `API.FORMS.CUSTOMER.R("member")` (shipped member create/edit).
  - `Forms.CUSTOMER_ORGANIZATION` — `API.FORMS.CUSTOMER.R("organization")` (shipped org settings upsert).
- **Form hooks (web-core)** — same ownership split as cpanel:
  - `useForm` reads an already-existing form entry,
  - `useShallowForm` injects a form reducer entry on mount (route/modal/scoped forms),
  - the screen owns layout, `sub` selection, duplicate-submit guards (`submittingRef`), read-on-edit hydration, and after-success behavior,
  - field hooks own value/errors; submit handlers send only the `sub`,
  - temporary private `useShallowForm` forms use `removeOnExit: true` by default; create drafts that must survive SPA nav use stable `formIdentify` + `removeOnExit: false`.
- **Read-on-edit**: edit routes pass only identity; edit mode hydrates via a `read` sub on entry. Show `Loadable` during injection + edit `read` in flight. Settings-like single-path forms (organization) always `read` on enter; after successful upsert they full-`redirect` to `CustomerHome` (see `flow-customer-organization.md`).
- **Multi-path create+update**: one route identify with `path: { create, update }` — `.cursor/rules/website-multi-path-form-routes.mdc`. Typed href builders live under `resources/configs/customer/formRoute.ts`. Single-path settings forms do **not** use multi-path.
- **Customer form screens**:
  - auth: `/login`, `/register`, `/reset-password` — `API.FORMS.R("auth")`; see `flow-auth.md`,
  - member form: `/customer/members/form` (+ `/:id`) — `Forms.CUSTOMER_MEMBER`; see `flow-customer-members.md` §5,
  - organization settings: `/customer/organization` — `Forms.CUSTOMER_ORGANIZATION`; see `flow-customer-organization.md`,
  - account settings: `/customer/settings` — planned (`Forms.CUSTOMER_SETTINGS` not registered yet); see `flow-settings.md`,
  - notification delete-all — see `flow-notifications.md`.
- **Shared form field surfaces** under `src/app/ui/components/form/`:
  - `FormTextField`, `FormActionButton`, `FormAvatarField`, `FormColorField`, `FormInputWrapper`, `FormProvider`.
  - Compose from `Utils` + semantic theme tokens.

## 3) Field surface contracts (shipped)

### 3.1 `FormTextField`

- Controlled via `useFormInput({ name })`.
- Native direction follows the page locale (**do not** set Utils `ltr` / `ta_l` on the input). Use logical `textAlign: "start"` so Arabic caret/typing works.
- Optional `suffix` — chrome after the input (string or node). When present, the input+suffix row is Utils `ltr` so the suffix stays on the **visual right** in both `ar` and `en`. The bare input (no suffix) still follows page direction.
- Rule: `.cursor/rules/website-form-text-field-direction.mdc`.
- Product convention for customer workspace forms: **no placeholders** unless the product owner explicitly asks.

### 3.2 `FormInputWrapper`

- Title: `variant="inputTitle"`.
- Subtitle: `variant="caption"` + `semanticColor.textTertiary` (must read quieter than the title).
- Errors: `semanticColor.stateError`.

### 3.3 `FormAvatarField`

- Form field name: optional `name` prop, default **`avatar_file`**. Organization settings passes `name="logo_file"`.
- Value is the uploaded filename string (empty string clears on backend update when sent).
- Immediate multipart: `uploadEntityMediaFile` → `API.ACTIONS.MULTIPART_UPLOAD` → `uploadedName`.
- URI helpers: `mediaStaticRoot` / `mediaImageUri` (`website/src/app/helpers/media.ts`).
- Preview priority: local object URL → form field value → optional `currentAvatarUrl`.
- Visual: compact navy identity circle (`primaryActionBackground` / `FiUser` / absolute `cover` image fill) + quiet accent text action — Ejtmaa identity language (same family as `IdentityAvatar`), not Masdaria camera-chip / stretched input-shell chrome.
- Rule: `.cursor/rules/website-form-avatar-field.mdc`.

### 3.4 `FormColorField`

- Controlled via `useFormInput({ name })`.
- Native `input type="color"` swatch + hex text in one `FormInputWrapper` row.
- Optional `fallback` hex for the picker when the form value is empty/invalid (org form uses `BrandColors.navy` / `BrandColors.orange`).
- W29: `baseCssStyle` holds only `...ElementStyles.inputReset`; behavior/pseudo styles in `cssStyle`; padding via `p={0}` shorthand on the swatch.
- Rule: `.cursor/rules/website-form-color-field.mdc`.

### 3.5 Success toasts

Automatic via `ResMainMessageMiddleware` — do not re-toast in `afterSentSuccess`. Rule: `.cursor/rules/website-form-success-toast-automatic.mdc`. Canonical member form: `CustomerMemberFormScreen`. Organization form: after upsert → `useRouter().redirect` to `CustomerHome` (`flow-customer-organization.md`).

## 4) Verification checklist

- Website `forms.ts` registers identities storing the requester builder (not a preselected `sub`).
- `requesters.website.ts` mirrors backend `requesters.website.ts` unions (W18).
- Route-scoped forms use `useShallowForm` + `inheritedFormIdentify`; submit sends only the `sub`.
- `loading = !exist || !!formLoading`; `Loadable` during injection + edit `read` where applicable.
- Shared field surfaces live under `src/app/ui/components/form/`.
- Form success toasts are automatic — no duplicate manual toasts in `afterSentSuccess`.
- Multi-path forms use one identify + `formRoute` href builders.
- `yarn type-check` passes in `website/`.

## 5) Traceability (member + organization form slices)

| Path | Role |
|---|---|
| `website/src/resources/configs/store/forms.ts` | `CUSTOMER_MEMBER`, `CUSTOMER_ORGANIZATION` |
| `website/src/resources/configs/customer/formRoute.ts` | `buildCustomerMemberFormHref` |
| `website/src/types/requesters/requesters.website.ts` | `customer.member`, `customer.organization` |
| `website/src/app/ui/components/form/FormAvatarField.tsx` | avatar / logo field |
| `website/src/app/ui/components/form/FormColorField.tsx` | color field |
| `website/src/app/helpers/entityMedia.ts` / `media.ts` | upload + URI |
| `website/src/app/ui/components/form/FormTextField.tsx` | direction + optional `suffix` |
| `website/src/resources/configs/urls.ts` | `ORG_PUBLIC_DOMAIN` for org subdomain suffix |
| `website/src/app/ui/components/form/FormInputWrapper.tsx` | subtitle color |
| `docs/platforms/website/flow-customer-members.md` | end-to-end member UI |
| `docs/platforms/website/flow-customer-organization.md` | end-to-end organization UI |
| `docs/platforms/backend/contracts/member-domain.md` §9 | member requester |
| `docs/platforms/backend/contracts/organization-domain.md` §9 | organization requester |

## 6) Related

- `docs/platforms/website/flow-auth.md`
- `docs/platforms/website/flow-customer-members.md`
- `docs/platforms/website/flow-customer-organization.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-notifications.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/invariants/website.md` (W11, W18)
- `.cursor/rules/website-multi-path-form-routes.mdc`
- `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`
- `.cursor/rules/website-form-avatar-field.mdc`
- `.cursor/rules/website-form-color-field.mdc`
- `.cursor/skills/website-customer-member-form/SKILL.md`
- `.cursor/skills/website-customer-organization-form/SKILL.md`
