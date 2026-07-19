# Website Form Foundation (Customer)

Customer portal contract (see `overview.md`).

## 1) Scope

Web-native requester form foundation for `website/`: the typed requester route map, shared `FORMS` identities, route-scoped create/edit form screens, shared form field surfaces, and the `useForm` / `useShallowForm` / `FormInput` hook contract — all on the website web-core form stack.

## 2) Web-native approach (governance)

- **Requester endpoint typing** — `website/src/types/requesters/requesters.website.ts` (shipped):
  - `visitor.auth`: `registerCustomer` | `login` | `resetPassword`,
  - `customer.customer`: `readSettings` | `updateSettings`,
  - `customer.member`: `read` | `create` | `update` | `delete`,
  - `customer.messageChannel`: `read` | `create` | `update` | `delete`,
  - `customer.messageTemplate`: `read` | `create` | `update` | `delete`,
  - `customer.organization`: `read` | `upsert`,
  - `customer.notification`: `deleteAll`,
  - `customer.subscription`: `subscribe`.
  - `customer.meeting`: `create` | `read` | `update` | `delete` | `approve` | participant/agenda/decision child subs (see `requesters.website.ts`).
  - Planned subs not yet shipped: `visitor.auth` `loginSocial` | `registerCustomerSocial` (Google social).
  - Endpoint builders (shipped): `API.FORMS.R("auth")(sub)` (visitor), `API.FORMS.CUSTOMER.R(requester)(sub)`.
- **Shared `FORMS` registrations** — `website/src/resources/configs/store/forms.ts` stores the requester endpoint builder (not a preselected `sub`); callers pass the truthful `sub` at send time.
  - `Forms.FORM1` — empty inherit for mockups / local tools.
  - `Forms.CUSTOMER_MEMBER` — `API.FORMS.CUSTOMER.R("member")` (shipped member create/edit).
  - `Forms.CUSTOMER_MESSAGE_CHANNEL` — `API.FORMS.CUSTOMER.R("messageChannel")` (shipped channel create/edit).
  - `Forms.CUSTOMER_MESSAGE_TEMPLATE` — `API.FORMS.CUSTOMER.R("messageTemplate")` (shipped template create/edit).
  - `Forms.CUSTOMER_ORGANIZATION` — `API.FORMS.CUSTOMER.R("organization")` (shipped org settings upsert).
  - `Forms.CUSTOMER_MEETING` — `API.FORMS.CUSTOMER.R("meeting")` (create form + details roadmap writes).
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
  - message-channel form: `/customer/message-channels/form` (+ `/:id`) — `Forms.CUSTOMER_MESSAGE_CHANNEL`; see `flow-customer-message-channels.md`,
  - message-template form: `/customer/message-templates/form` (+ `/:id`) — `Forms.CUSTOMER_MESSAGE_TEMPLATE`; see `flow-customer-message-templates.md`,
  - meeting form: `/customer/meetings/form` — `Forms.CUSTOMER_MEETING` create-only path; details roadmap uses the same form identity for modal/page writes — see `flow-customer-meetings.md`,
  - organization settings: `/customer/organization` — `Forms.CUSTOMER_ORGANIZATION`; see `flow-customer-organization.md`,
  - account settings: `/customer/settings` — planned (`Forms.CUSTOMER_SETTINGS` not registered yet); see `flow-settings.md`,
  - notification delete-all — see `flow-notifications.md`.
- **Shared form field surfaces** under `src/app/ui/components/form/`:
  - `FormTextField`, `FormActionButton`, `FormAvatarField`, `FormColorField`, `FormChoiceField`, `FormEntityPickerField`, `FormDateTimeField`, `FormInputWrapper`, `FormProvider`,
  - template helpers: `FormTemplateVariableInsert`, `FormMessageTemplateVariablesField` (see §3.2c; consumer `flow-customer-message-templates.md`),
  - Compose from `Utils` + semantic theme tokens.
  - Modals: `ENTITY_PICKER` (`openEntityPicker`), `DATETIME_PICKER` (`openDateTimePicker`), `CONFIRM` (`confirm`), `FORM` (`openForm` thin chrome shell) via `ModalBase` / `ModalsManager` — registry `resources/configs/store/modals.ts`.
  - **`FORM` modal bodies are real forms** — each body owns `useShallowForm` + `FormProvider` + shared `Form*` fields + `d.send({ sub })` exactly like route screens (`CustomerMeetingFormScreen` / `CustomerMemberFormScreen`). Do **not** share the parent page form context, raw `<input>` state, or `setValues`+custom chrome inside the modal. `FormModal` / `openForm` only supply title/subtitle/shell; field errors, loading, and submit stay on the form stack. Canonical meeting examples: `MeetingBasicsModalForm`, `MeetingParticipantAddModalForm`, `MeetingSubjectModalForm`.

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
- Optional `actionArea` — beside title (legacy/rare).
- Optional `headerArea` — **above** the input chrome (e.g. template placeholder insert chips). Prefer this over packing tools into `actionArea` next to the title.
- Optional `bareField` — no pad/border/fill chrome (choice lists, Pro variables table).
- Optional field shorthands: `fieldMinH`, `fieldPt`, `fieldPb`, `fieldAi_fs`; `fieldCssStyle` only for non-shorthand escapes.

### 3.2b `FormTextField`

- Supports `headerArea` / `actionArea` forwarded to the wrapper.
- Native input/textarea uses `ElementStyles.inputReset` in `baseCssStyle` (not `buttonReset`).
- Multiline chrome via wrapper `fieldMinH` / `fieldPt` / `fieldPb` / `fieldAi_fs`.

### 3.2c Template placeholder helpers (message templates)

| Component | Role |
|---|---|
| `FormTemplateVariableInsert` | Human-labeled chips; inserts `{{key}}` into named text field |
| `FormMessageTemplateVariablesField` | Fixed Pro variable → slot number rows; seeds defaults `1`–`4` |
| `messageTemplatePlaceholders.ts` | Shared keys + i18n label keys + `normalizeMessageTemplateVariables` |

UX locks: chip **labels** are human text (tokens only after insert into the field value); chips use `headerArea` above the input, not `actionArea` beside the title; Pro table uses “variable number” wording (not “Meta slot”). Full consumer: `flow-customer-message-templates.md` §6.

List these under shared form surfaces in §1 when adding more template helpers.

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

### 3.5 `FormChoiceField`

**File:** `website/src/app/ui/components/form/FormChoiceField.tsx`

| Concern | Contract |
|---|---|
| Control | `useFormInput({ name })` |
| Purpose | Discrete **form** values as equal-width tiles (e.g. meeting `type`) |
| Not for | List/history filters — those use `FilterOptionChip(s)` (underline tabs) |
| Selected | Soft accent fill (`canvasAccentSoftBackground`) + accent border + `FiCheck` |
| Unselected | `inputBackground` + `inputBorder` |
| Read-only | When `!setValue` **or** `readOnly` prop: `opc` muted + `cursor: not-allowed` + `disabled` |
| Wrapper | `FormInputWrapper` — default chrome via Utils shorthands; `bareField` for chrome-less lists/tables; optional `fieldMinH` / `fieldPt` / `fieldPb` / `fieldAi_fs`; `fieldCssStyle` only for non-shorthand escapes |
| Click | `extra.onClick` → `setValue(option.value)` |
| Value normalize | `choiceFieldValue(value)` — string or SelectOption `{ value }` → string (shared with conditional form branching) |
| Read hydrate | Backend `read` should return SelectOption via `toEnumForSelect` for enum-backed choices — `docs/platforms/backend/patterns/requester-read-select-hydrate.md` |

Canonical consumers: `CustomerMeetingFormScreen` type field; `CustomerMessageChannelFormScreen` / `CustomerMessageTemplateFormScreen` type (create editable / update `readOnly`).

### 3.6 `FormEntityPickerField` + `ENTITY_PICKER`

**Files:**

- Field: `website/src/app/ui/components/form/FormEntityPickerField.tsx`
- Modal: `website/src/app/ui/components/modals/EntityPickerModal.tsx`
- Wrapper: `SelectableEntityCard.tsx`
- Types: `modals/entity-picker/types.ts`
- Registry: `modals/entity-picker/configs/index.ts` + one file per `ident`
- Shipped config: `configs/members.tsx` (`listable: "members"`, GQL `EntityPickerMembers`, `CustomerMemberCard` + **`selected`**)
- Shipped config: `configs/messageChannels.tsx` (`listable: "messageChannels"`, `searchable: false`, filter `status: ACTIVE` + optional `type`, `CustomerMessageChannelCard` + **`selected`** — same selection chrome as members)

#### Field

| Concern | Contract |
|---|---|
| Open | `openEntityPicker({ ident, title, multi?, initValue, filter?, … })` |
| Single store | `{ value, label, avatarUrl? }` or `""` when cleared |
| Multi store | `EntityPickerSelection[]` |
| Read hydrate | Backend `read` should return `{ value, label }` via related `model.forSelect(lang)` — `docs/platforms/backend/patterns/requester-read-select-hydrate.md` |
| Chrome | Chips with `IdentityAvatar` + label; empty uses `emptyLabel`; pick action button |
| Read-only | When `!setValue` |

#### Modal

| Concern | Contract |
|---|---|
| Adapter id | `entity-picker-${ident}` inherit `config.inheritedAdapterIdentify` |
| Search | Optional: `searchable` (config/prop, default true). When on: draft + Enter → `committedSearch` → `buildFilter`. When off: no `SearchField` |
| Filter extras | `filter` prop → `buildFilter(search, extras)` (e.g. channel `type`) |
| List scroll | `Col` `minH` when empty/busy + `maxH` cap + `customScroll` (not `ovr_y`) so `Empty` overlay has height |
| Pagination | Map `thereMoreRecords`; `LoadMoreButton`; `mLoad({ query })` without reload |
| Selection | Single replaces; multi toggles; `SelectableEntityCard` + config `Card` |
| Confirm | Passes full selection meta including `avatarUrl` |
| Escape / route change | Closes modal |
| i18n | `ui.modals.entityPicker.*` |

Skill: `.cursor/skills/website-entity-picker/SKILL.md`. Scroll: `.cursor/rules/website-custom-scroll-contract.mdc`.

Canonical consumers: meeting chairperson (`ident="members"`); template channel (`ident="messageChannels"`).

### 3.7 `FormDateTimeField` + `DATETIME_PICKER`

**Files:**

- Field: `website/src/app/ui/components/form/FormDateTimeField.tsx`
- Modal: `website/src/app/ui/components/modals/DateTimePickerModal.tsx`
- Theme helper: `website/src/resources/emotion/styles/datepicker.ts` → `datepickerTheme`
- Library CSS: `react-datepicker/dist/react-datepicker.css` via `dependencies.scss` only

#### Field

| Concern | Contract |
|---|---|
| Value | ISO string in form reducer |
| Chrome | Date chip + time chip (or `emptyLabel`) + pick action |
| Open | `openDateTimePicker({ title, initValue, minDate?, pastDateError })` |
| After pick | Reject if `<= Date.now()` via `input.set(iso, [pastDateError])`; else `set(iso, [])` |

#### Modal

| Concern | Contract |
|---|---|
| Calendar | `react-datepicker` **inline**; class `ejt-datepicker` (+ `--rtl` for `ar`) |
| Layout | Calendar **full width** of modal (flex weeks/days); not compact centered inline-block |
| Time | 15-minute chip grid; scroll Col `minH={0}` + `maxH` + `customScroll` |
| Theme | `getColor(semanticColor…)` → `datepickerTheme({…})` on calendar wrapper `cssStyle` |
| Confirm | Returns ISO via `onSelect`; cancel / Escape closes |
| Light selected day | `primaryActionBackground` / `primaryActionText` |
| Dark selected day | `accentActionBackground` / `accentActionText` |
| i18n | `ui.modals.dateTimePicker.*` |

Rule: `.cursor/rules/website-third-party-widget-emotion-theme.mdc`. **Forbidden:** brand-hex SCSS overlay (deleted `react-datepicker-theme.scss`).

Canonical consumer: meeting `datetime`.

### 3.8 `ConfirmModal`

**File:** `website/src/app/ui/components/modals/ConfirmModal.tsx`

| Concern | Contract |
|---|---|
| API | `await confirm(description, tone?)` → `Promise<boolean>` |
| Tone | `"danger"` (default) or `"primary"` — maps to `FormActionButton` |
| Chrome | Same card shell as other modals: `semanticColor.cardBackground`, border, radius, shadow |
| Confirm | `closeMe({ onClosed: onConfirm })` then resolve `true` (WILL_CLOSE replaces `onClosed`) |
| Cancel / backdrop | resolve `false` |
| Registry | `Modals.CONFIRM` / `cancelable: true` in `resources/configs/store/modals.ts` |
| i18n | `ui.modals.confirm.*` |
| Forbidden | `window.confirm` / `window.prompt` for product confirms |

Canonical consumers: member + message-channel form delete. Rule: `.cursor/rules/website-confirm-modal.mdc`.

### 3.9 `FormActionButton` tones

`tone`: `primary` | `neutral` | `secondary` | `danger`.

- `danger` uses `semanticColor.dangerActionBackground` + primary-on-fill text (confirm modal confirm action).
- Native `<button>` always filled via `bg` shorthand + `baseCssStyle={{...ElementStyles.buttonReset}}`; hover/cursor in `cssStyle` (`extra.onClick`).

### 3.10 Multi-action form loading

When a screen has Save + Delete (or multiple `sub` sends on one form):

- Map `saving` = `status === "SENDING" && (currentSub === "create" \|\| currentSub === "update")`
- Map `deleting` = `status === "SENDING" && currentSub === "delete"`
- Pass `loading={saving}` / `loading={deleting}` on the matching `FormActionButton`
- Both stay `disabled` while any `formLoading` / `!exist`
- Do **not** bind Save `loading` to bare `status === "SENDING"` (that spins Save during delete)

Canonical: `CustomerMemberFormScreen`, `CustomerMessageChannelFormScreen`.

### 3.11 Success toasts

Automatic via `ResMainMessageMiddleware` — do not re-toast in `afterSentSuccess`. Rule: `.cursor/rules/website-form-success-toast-automatic.mdc`. Canonical member form: `CustomerMemberFormScreen`. Organization form: after upsert → `useRouter().redirect` to `CustomerHome` (`flow-customer-organization.md`). Meeting create: after success → `d.reset()` then `CustomerMeetingDetails` with `replace: true` (`flow-customer-meetings.md`).

## 4) Verification checklist

- Website `forms.ts` registers identities storing the requester builder (not a preselected `sub`).
- `requesters.website.ts` mirrors backend `requesters.website.ts` unions (W18).
- Route-scoped forms use `useShallowForm` + `inheritedFormIdentify`; submit sends only the `sub`.
- `loading = !exist || !!formLoading`; `Loadable` during injection + edit `read` where applicable.
- Shared field surfaces live under `src/app/ui/components/form/`.
- Form success toasts are automatic — no duplicate manual toasts in `afterSentSuccess`.
- Multi-path forms use one identify + `formRoute` href builders.
- Destructive confirms use `confirm()` (`CONFIRM` modal), never `window.confirm`.
- Multi-action forms scope button `loading` by `currentSub` (§3.10).
- `yarn type-check` passes in `website/`.

## 5) Traceability (form foundation — shared surfaces)

| Path | Role |
|---|---|
| `website/src/resources/configs/store/forms.ts` | `CUSTOMER_MEMBER`, `CUSTOMER_ORGANIZATION`, `CUSTOMER_MEETING`, `CUSTOMER_MESSAGE_CHANNEL`, `CUSTOMER_MESSAGE_TEMPLATE` |
| `website/src/resources/configs/customer/formRoute.ts` | `buildCustomerMemberFormHref`, `buildCustomerMeetingFormHref`, `buildCustomerMessageChannelFormHref`, `buildCustomerMessageTemplateFormHref` |
| `website/src/types/requesters/requesters.website.ts` | `customer.member`, `customer.messageChannel`, `customer.messageTemplate`, `customer.organization`, `customer.meeting` |
| `website/src/app/ui/components/form/FormInputWrapper.tsx` | `headerArea`, `bareField`, field shorthands |
| `website/src/app/ui/components/form/FormTextField.tsx` | `headerArea` / `inputReset` / field chrome |
| `website/src/app/ui/components/form/FormTemplateVariableInsert.tsx` | human-label insert chips (§3.2c) |
| `website/src/app/ui/components/form/FormMessageTemplateVariablesField.tsx` | Pro variable → number rows (§3.2c) |
| `website/src/resources/configs/customer/messageTemplatePlaceholders.ts` | placeholder keys + normalize |
| `website/src/app/ui/components/form/FormAvatarField.tsx` | avatar / logo field |
| `website/src/app/ui/components/form/FormColorField.tsx` | color field |
| `website/src/app/ui/components/form/FormChoiceField.tsx` | discrete form choice tiles |
| `website/src/app/ui/components/form/FormEntityPickerField.tsx` | entity picker field → modal |
| `website/src/app/ui/components/modals/EntityPickerModal.tsx` | `ENTITY_PICKER` shell (search + customScroll + loadMore) |
| `website/src/app/ui/components/modals/SelectableEntityCard.tsx` | selectable wrapper around page cards |
| `website/src/app/ui/components/modals/entity-picker/configs/` | per-entity gql + Card (`members.tsx`, `messageChannels.tsx`, …) |
| `website/src/app/ui/components/modals/entity-picker/configs/index.ts` | registry of idents |
| `website/src/app/ui/components/modals/entity-picker/types.ts` | selection shape (`avatarUrl`, `searchable?`) |
| `website/src/resources/configs/store/modals.ts` | `ENTITY_PICKER`, `DATETIME_PICKER`, `CONFIRM`, `FORM` |
| `website/src/app/ui/components/modals/FormModal.tsx` | Thin `FORM` chrome + `openForm` (body owns form stack) |
| `website/src/app/ui/components/customer/meetings/MeetingBasicsModalForm.tsx` | Meeting basics modal form (`read`/`update`) |
| `website/src/app/ui/components/customer/meetings/MeetingParticipantAddModalForm.tsx` | Add participant modal form |
| `website/src/app/ui/components/customer/meetings/MeetingSubjectModalForm.tsx` | Agenda/decision subject modal form |
| `website/src/app/ui/components/form/FormDateTimeField.tsx` | datetime field → `DATETIME_PICKER` modal |
| `website/src/app/ui/components/modals/DateTimePickerModal.tsx` | datetime modal shell |
| `website/src/app/ui/components/modals/ConfirmModal.tsx` | `CONFIRM` shell + `confirm()` helper |
| `website/src/resources/emotion/styles/datepicker.ts` | `datepickerTheme` (`.ejt-datepicker`) |
| `website/src/resources/emotion/styles/scroll.ts` | `customScroll` helper |
| `website/src/app/helpers/entityMedia.ts` / `media.ts` | upload + URI |
| `website/src/app/ui/components/form/FormTextField.tsx` | direction + optional `suffix` |
| `website/src/resources/configs/urls.ts` | `ORG_PUBLIC_DOMAIN` for org subdomain suffix |
| `website/src/app/ui/components/form/FormInputWrapper.tsx` | subtitle color |
| `docs/platforms/website/flow-customer-members.md` | end-to-end member UI |
| `docs/platforms/website/flow-customer-message-channels.md` | end-to-end message-channel UI |
| `docs/platforms/website/flow-customer-organization.md` | end-to-end organization UI |
| `docs/platforms/website/flow-customer-meetings.md` | end-to-end meetings UI |
| `docs/platforms/backend/contracts/member-domain.md` §9 | member requester |
| `docs/platforms/backend/contracts/message-channel-domain.md` §5 | message-channel requester |
| `docs/platforms/backend/contracts/organization-domain.md` §9 | organization requester |
| `docs/platforms/backend/contracts/meeting-domain.md` | meeting requester + filter |

## 6) Related

- `docs/platforms/website/flow-auth.md`
- `docs/platforms/website/flow-customer-members.md`
- `docs/platforms/website/flow-customer-message-channels.md`
- `docs/platforms/website/flow-customer-organization.md`
- `docs/platforms/website/flow-customer-meetings.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-notifications.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/invariants/website.md` (W11, W18)
- `.cursor/rules/website-multi-path-form-routes.mdc`
- `.cursor/rules/website-confirm-modal.mdc`
- `.cursor/rules/requester-read-no-entity-id-echo.mdc`
- `docs/platforms/backend/patterns/requester-read-select-hydrate.md`
- `.cursor/rules/requester-read-select-hydrate.mdc`
- `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`
- `.cursor/skills/website-confirm-modal/SKILL.md`
- `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`
- `.cursor/rules/website-form-avatar-field.mdc`
- `.cursor/rules/website-form-color-field.mdc`
- `.cursor/rules/website-third-party-widget-emotion-theme.mdc`
- `.cursor/rules/website-custom-scroll-contract.mdc`
- `.cursor/skills/website-customer-member-form/SKILL.md`
- `.cursor/skills/website-customer-message-channels/SKILL.md`
- `.cursor/skills/website-confirm-modal/SKILL.md`
- `.cursor/skills/website-customer-organization-form/SKILL.md`
- `.cursor/skills/website-customer-meeting-form/SKILL.md`
- `.cursor/skills/website-entity-picker/SKILL.md`
