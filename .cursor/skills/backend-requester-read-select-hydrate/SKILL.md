---
name: backend-requester-read-select-hydrate
description: >-
  Shapes backend requester read values as SelectOption for website FormChoiceField
  and FormEntityPickerField using RequesterBase.toEnumForSelect and per-model
  forSelect(lang). Use when adding or fixing requester read hydration, enum choice
  fields, entity-picker refs on edit forms, MessageChannel/MessageTemplate read,
  or forSelect on ORM models used as form refs.
---

# Backend requester read — select hydrate

## When to Use

- Adding or fixing a requester `read` that hydrates a website edit form.
- `read` returns a domain enum bound to `FormChoiceField`.
- `read` returns a related entity bound to `FormEntityPickerField`.
- Adding `forSelect(lang)` on an ORM model used as a form/entity-picker ref.
- Reviewing MessageChannel / MessageTemplate (or similar) read payloads for select shape.

## Instructions

1. Read `docs/platforms/backend/patterns/requester-read-select-hydrate.md` and `.cursor/rules/requester-read-select-hydrate.mdc`.
2. Also keep `.cursor/rules/requester-read-no-entity-id-echo.mdc` — identity stays in form `initProps`, not `read` values.
3. Inventory `read` fields that bind to choice tiles or entity pickers.
4. **Enums:** use `await this.toEnumForSelect(row.get("…"), "enumKey")` where `enumKey` exists under `general.enums` (ar/en). Type `ReadResult` as `SelectOption`.
5. **Entity refs:** if missing, add `forSelect(_lang: string): SelectOption` on the related model (`value` = id, `label` = display). Call `model?.forSelect(this.context.lang()) ?? null` from `read`.
6. Do not hand-build SelectOption in the requester when the helper/method exists.
7. Leave write path on `joi.select({ validValues })` / Opt helpers — do not narrow writes to SelectOption-only.
8. Confirm website consumers are `FormChoiceField` / `FormEntityPickerField` (already SelectOption-tolerant) — see `docs/platforms/website/flow-form-foundation.md` §3.5–3.6.
9. Verify: backend `yarn type-check`.

## Canonical shipped examples

- `MessageChannelModel.forSelect`
- `MessageChannelRequester.read` → `type` via `toEnumForSelect(..., "messageChannelType")`
- `MessageTemplateRequester.read` → `type` via `toEnumForSelect(..., "messageTemplateType")` + `messageChannel` via `forSelect`

## Related

- `.cursor/skills/backend-requester-governance/SKILL.md` (write-path `joi.select`)
- `.cursor/skills/orm-model-generator/SKILL.md` (when adding model helpers)
- `.cursor/skills/website-entity-picker/SKILL.md` (picker store shape)
- Domain contracts: `docs/platforms/backend/contracts/message-channel-domain.md`, `message-template-domain.md`
