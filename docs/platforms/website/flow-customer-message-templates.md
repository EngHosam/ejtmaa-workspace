# Website Flow — Customer Message Templates

Authenticated customer org message-template directory + multi-path form on `CUSTOMER_MAIN`. Shell/breadcrumb: `flow-customer-shell.md` §7.1. Backend: `docs/platforms/backend/contracts/message-template-domain.md`. Form foundation: `flow-form-foundation.md` (§3.2 wrapper, §3.5 choice, §3.6 picker, §3.8 ConfirmModal).

## 1) Scope

### Shipped

- Directory (GQL list + ResultLane load-more) with Add + card Edit.
- Multi-path form `CustomerMessageTemplateForm` (create/update/delete) via `Forms.CUSTOMER_MESSAGE_TEMPLATE` → `messageTemplate` requester.
- Drawer tile `itemMessageTemplates` / `CustomerMessageTemplates` / **`FiMessageSquare`** (same glyph on list card).
- Type chooser: create editable / update `readOnly` (`FormChoiceField` + `choiceFieldValue`).
- Fields by kind:

| Kind | Channel | Content UI |
|---|---|---|
| `EJTMAA_EMAIL` | none | subject + body + insert chips |
| `CUSTOM_EMAIL` | picker required | subject + body + insert chips |
| `ADWHATS` | picker required | body + insert chips |
| `ADWHATS_PRO` | picker required | `meta_template_id` + variables table |

- Placeholder chips: human labels only (`FormTemplateVariableInsert` in `headerArea` **above** the input). Stored tokens remain `{{memberName}}` etc. inside the text value.
- `ADWHATS_PRO` variables: fixed four meeting-notify keys → editable slot numbers (default `1`–`4`); no raw `{{token}}` in the table UI (`FormMessageTemplateVariablesField`). Config: `messageTemplatePlaceholders.ts`.
- Channel picker `messageChannels`: `searchable: false`; filter always `status: ACTIVE` + `type` from template type; Card must forward `selected` like members.
- Delete: `confirm(…, "danger")` → `delete` → `nav.back()`.
- Loading: `saving` / `deleting` from `currentSub`.
- Create: stable `formIdentify` + `d.reset()` on success; changing type clears `messageChannel`.
- `read` must not echo `messageTemplate` id; hydrate `type` / `messageChannel` as SelectOption (`requester-read-select-hydrate.md`).

### Not shipped

- Template list search/filter (`_MessageTemplateFilter` absent).
- Seed templates; Meta language field; real send pipeline.
- cpanel / supervisor template UI.

### Product risks (ops / UX)

- Channel `testConnection()` stub often saves channels as `DISABLED` → ACTIVE-only picker may look empty.
- Deploy must sync DB column `message_templates.meta_template_id`.

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| List route | `CustomerMessageTemplates` → `/customer/message-templates` |
| Form route | `CustomerMessageTemplateForm` → `/customer/message-templates/form` (+ `/:id`) |
| List page | `website/src/app/ui/pages/customer/CustomerMessageTemplates.tsx` |
| Form page | `website/src/app/ui/pages/customer/CustomerMessageTemplateForm.tsx` |
| List screen | `…/message-templates/CustomerMessageTemplatesScreen.tsx` |
| Form screen | `…/message-templates/CustomerMessageTemplateFormScreen.tsx` |
| Card / skeleton | `CustomerMessageTemplateCard.tsx` / `MessageTemplateCardSkeleton.tsx` |
| Hook | `…/hooks/useCustomerMessageTemplates.ts` |
| Placeholders | `resources/configs/customer/messageTemplatePlaceholders.ts` |
| Insert chips | `form/FormTemplateVariableInsert.tsx` |
| Variables table | `form/FormMessageTemplateVariablesField.tsx` |
| Href | `buildCustomerMessageTemplateFormHref` |
| Form registry | `Forms.CUSTOMER_MESSAGE_TEMPLATE` → `API.FORMS.CUSTOMER.R("messageTemplate")` |
| Channel picker | `entity-picker/configs/messageChannels.tsx` |

List + form: `layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`. List breadcrumb parent `CustomerHome`; form parent `CustomerMessageTemplates`. Form identify **after** list on the same prefix (static-before-parametric).

## 3) Form contract

`formType = id ? "update" : "create"` from `useCurrentParams<"CustomerMessageTemplateForm">`.

| Mode | Behavior |
|---|---|
| create | Stable `formIdentify` (`customer-message-template-form-create`); **`d.reset()` on create success**; type change clears `messageChannel` |
| update | `initProps.values: { messageTemplate: id }`; `didEntered` → `read`; type `readOnly` |
| delete | `confirm` → `delete` → `nav.back()` |
| loading | `saving` = create\|update; `deleting` = delete |

## 4) Directory data adapter

Mount-private `"customer-message-templates"` inheriting `DATA_ADAPTERS.CUSTOMER_GQL`, `listable: "messageTemplates"`. Selection: `id`, `name`, `type`, `subject`, `messageChannel { id name }`, `total_count`.

## 5) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.messageTemplates.*` | list |
| `ui.pages.customer.messageTemplateForm.*` | form (incl. `var*`, `metaTemplateId*`, `variables*`) |
| `ui.layouts.customerMainLayout.drawer.itemMessageTemplates` | drawer |

## 6) Locked UI agreements (this slice)

| Agreement | Implementation |
|---|---|
| Chip **labels** are human text, not `{{tokens}}` | `FormTemplateVariableInsert` |
| Chips sit **above** the field, not beside the title | `FormInputWrapper.headerArea` |
| Pro variables table: human labels only; slot = “variable number” (not “Meta slot”) | `FormMessageTemplateVariablesField` + i18n |
| Pro requires Meta template id field | `meta_template_id` |
| Channel picker: ACTIVE only + matching type | `messageChannels` `buildFilter` |
| Picker card selection chrome matches members | `selected` prop on `CustomerMessageChannelCard` |
| Type-conditional Joi leftovers use `.strip()`; write with `?? null` (no `attrsForType`) | backend requesters |

## 7) Traceability map — website change set

| Path | Role | § |
|---|---|---|
| `src/app/ui/pages/customer/CustomerMessageTemplates.tsx` | List page shell | §2 |
| `src/app/ui/pages/customer/CustomerMessageTemplateForm.tsx` | Form page shell | §2 |
| `src/app/ui/components/customer/message-templates/CustomerMessageTemplatesScreen.tsx` | ResultLane directory | §1–§4 |
| `src/app/ui/components/customer/message-templates/CustomerMessageTemplateFormScreen.tsx` | Kind branching form | §3–§6 |
| `src/app/ui/components/customer/message-templates/CustomerMessageTemplateCard.tsx` | List card + `FiMessageSquare` | §1 |
| `src/app/ui/components/customer/message-templates/MessageTemplateCardSkeleton.tsx` | Loading skeleton | §1 |
| `src/app/ui/components/customer/hooks/useCustomerMessageTemplates.ts` | List adapter | §4 |
| `src/app/ui/components/form/FormTemplateVariableInsert.tsx` | Insert chips | §6 |
| `src/app/ui/components/form/FormMessageTemplateVariablesField.tsx` | Pro variables rows | §6 |
| `src/app/ui/components/form/FormTextField.tsx` | `headerArea` / `inputReset` / field chrome props | form foundation |
| `src/app/ui/components/form/FormInputWrapper.tsx` | `headerArea`, `bareField`, field shorthands | form foundation |
| `src/app/ui/components/form/FormChoiceField.tsx` | `readOnly` + `choiceFieldValue` + `bareField` | §3 |
| `src/app/ui/components/form/FormEntityPickerField.tsx` | `fieldMinH`; filter extras | §1 |
| `src/app/ui/components/modals/EntityPickerModal.tsx` | empty `minH`; searchable gate | §1 |
| `src/app/ui/components/modals/entity-picker/types.ts` | `searchable?`; Card `selected` | §1 |
| `src/app/ui/components/modals/entity-picker/configs/index.ts` | Register `messageChannels` | §2 |
| `src/app/ui/components/modals/entity-picker/configs/messageChannels.tsx` | ACTIVE+type filter; selected | §1 |
| `src/app/ui/components/modals/entity-picker/configs/members.tsx` | Canonical selected Card | related |
| `src/app/ui/components/customer/message-channels/CustomerMessageChannelCard.tsx` | `selected` chrome | §6 |
| `src/app/ui/components/customer/message-channels/CustomerMessageChannelFormScreen.tsx` | Choice `readOnly` align | related |
| `src/resources/configs/customer/messageTemplatePlaceholders.ts` | Keys + label i18n keys + normalize | §6 |
| `src/resources/configs/customer/formRoute.ts` | Template form href | §2 |
| `src/resources/configs/routes.ts` | Routes + `MPagesRoutes` | §2 |
| `src/app/ui/components/customer/CustomerDrawer.tsx` | Drawer tile `FiMessageSquare` | §1 / shell |
| `src/resources/configs/store/forms.ts` | `CUSTOMER_MESSAGE_TEMPLATE` | §2 |
| `src/resources/translations/ar.ts` / `en.ts` | List/form copy | §5 |
| `src/types/gql/definitions/customer.graphql` | Mirror SDL | backend §4 |
| `src/types/gql/gql-types/customer.ts` | Generated mirror | generated |
| `src/types/requesters/requesters.website.ts` | `messageTemplate` subs | backend §5b |
| `lib/tsconfig.tsbuildinfo` | Build cache | excluded from narrative |

## 8) Related

- `flow-customer-message-channels.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/patterns/requester-read-select-hydrate.md`
- `.cursor/skills/website-customer-message-templates/SKILL.md`
- `.cursor/skills/website-entity-picker/SKILL.md`
- `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`
- `.cursor/rules/message-template-domain.mdc`
- `.cursor/rules/requester-type-conditional-strip.mdc`
- `.cursor/rules/website-message-template-form-ux.mdc`
