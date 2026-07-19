---
name: website-confirm-modal
description: >-
  Adds or wires the website CONFIRM modal (ConfirmModal + confirm() helper) for
  destructive or irreversible actions. Use when replacing window.confirm, adding
  delete confirms, or registering CONFIRM in modals.ts.
---

# Website confirm modal

## When to Use

- Adding a delete / irreversible confirm on website customer UI.
- Replacing `window.confirm`.
- Changing `ConfirmModal` chrome, tones, or i18n.

## Instructions

1. Read `docs/platforms/website/flow-form-foundation.md` §3.8 and `.cursor/rules/website-confirm-modal.mdc`.
2. Ensure `Modals.CONFIRM` is registered in `resources/configs/store/modals.ts` with `ConfirmModal` and `cancelable: true`.
3. Call `await confirm(description, "danger" | "primary")` — default tone is `"danger"`.
4. On confirm success path, run the requester `sub` (e.g. `delete`) only after `true`.
5. Keep modal chrome on semantic card tokens (same family as EntityPicker / DateTimePicker).
6. i18n under `ui.modals.confirm.*` (ar/en). Description copy stays on the page translator.
7. For multi-action forms, pair with per-`currentSub` button loading (`flow-form-foundation.md` §3.10).
8. Verify: website `yarn type-check`.

## Related

- `.cursor/skills/website-customer-member-form/SKILL.md`
- `.cursor/skills/website-customer-message-channels/SKILL.md`
- `website/src/app/ui/components/modals/DateTimePickerModal.tsx` (`openDateTimePicker` promise pattern)
