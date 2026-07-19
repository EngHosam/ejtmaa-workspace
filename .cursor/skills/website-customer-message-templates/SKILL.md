---
name: website-customer-message-templates
description: >-
  Ships or extends the website customer message-templates directory and
  multi-path form (kinds, channel picker, ConfirmModal delete, FiMessageSquare
  drawer). Use when changing template list/form UI, messageChannels entity
  picker, or drawer tile for CustomerMessageTemplates.
---

# Website customer message templates

## When to Use

- Adding or changing `/customer/message-templates` list or form UI.
- Wiring template kind branching or `messageChannels` entity picker.
- Aligning drawer `CustomerMessageTemplates` / `FiMessageSquare` with the list card.

## Instructions

1. Read `docs/platforms/website/flow-customer-message-templates.md` and `docs/platforms/backend/contracts/message-template-domain.md` (locked agreements §6 / write strip §5b).
2. Confirm backend `MessageTemplateRequester` + `MESSAGE_TEMPLATE` ability + `customer.messageTemplate` map. Unused kind fields: `.strip()` + flat `?? null` — `.cursor/rules/requester-type-conditional-strip.mdc`.
3. List `CustomerMessageTemplates` on `CUSTOMER_MAIN`; form multi-path after list identify — `.cursor/rules/website-multi-path-form-routes.mdc`.
4. Drawer: identify `CustomerMessageTemplates`, glyph **`FiMessageSquare`** — same on card.
5. Register `Forms.CUSTOMER_MESSAGE_TEMPLATE` → `API.FORMS.CUSTOMER.R("messageTemplate")`. Href: `buildCustomerMessageTemplateFormHref`.
6. Adapter: `"customer-message-templates"` inherit `CUSTOMER_GQL`, `listable: "messageTemplates"`. No history search until `_MessageTemplateFilter` exists.
7. Form kinds: `EJTMAA_EMAIL` (subject+body); `CUSTOM_EMAIL`/`ADWHATS`/`ADWHATS_PRO` (+ channel); Pro requires `meta_template_id` + `FormMessageTemplateVariablesField`. Insert chips via `headerArea` (human labels). Placeholders config: `messageTemplatePlaceholders.ts`. Type `readOnly` on update. Clear channel when create type changes.
8. Channel picker: `messageChannels.tsx` — `searchable: false`; always `status: ACTIVE` + `type`; **pass `selected` into `CustomerMessageChannelCard`** (same as members).
9. Delete: `confirm(…, "danger")`. Loading: `saving` / `deleting` from `currentSub`.
10. Create: stable `formIdentify` + `d.reset()` on success. No `messageTemplate` id echo from `read`. Hydrate via select pattern.
11. Update flow docs + verify website `yarn type-check`.

## Related

- `.cursor/rules/website-message-template-form-ux.mdc`
- `.cursor/skills/website-customer-message-channels/SKILL.md`
- `.cursor/skills/website-entity-picker/SKILL.md`
- `.cursor/skills/website-confirm-modal/SKILL.md`
- `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`
- `.cursor/rules/requester-type-conditional-strip.mdc`
- `.cursor/rules/message-template-domain.mdc`
