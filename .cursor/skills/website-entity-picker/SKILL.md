---
name: website-entity-picker
description: >-
  Ships or extends website ENTITY_PICKER configs and FormEntityPickerField
  (config registry, customScroll list, loadMore, avatarUrl persistence). Use when
  adding a new picker ident or changing entity-picker modal/field behavior.
---

# Website entity picker

## When to Use

- Adding a new `ENTITY_PICKER` ident (beyond `members`).
- Changing picker search, selection, load-more, scroll, or field chrome (`avatarUrl`).
- Wiring `FormEntityPickerField` to a form that needs org-owned entity refs.

## Instructions

1. Add `modals/entity-picker/configs/<ident>.tsx` with `gql`, `listable`, `buildFilter(search)`, `Card`, `getValue` / `getLabel` / optional `getAvatarUrl`. Register in `configs/index.ts`. Types: `entity-picker/types.ts`.
2. Prefer the **same page Card** inside `SelectableEntityCard` (pass `selected` when the card supports it).
3. Modal (`EntityPickerModal`): Enter-committed text search only; list Col `minH={0}` + `customScroll`; `LoadMoreButton` when `thereMoreRecords` — `.cursor/rules/website-custom-scroll-contract.mdc`.
4. Field (`FormEntityPickerField`): single value `{ value, label, avatarUrl? }`; multi = array; show `IdentityAvatar` in chrome when `avatarUrl` is set.
5. Register modal in `resources/configs/store/modals.ts` as `ENTITY_PICKER` if not already present. i18n under `ui.modals.entityPicker.*`.
6. Update `docs/platforms/website/flow-form-foundation.md` §3.6 (+ consumer flow doc). Verify `yarn type-check` in `website/`.

## Canonical reference

- Modal: `website/src/app/ui/components/modals/EntityPickerModal.tsx`
- Config: `website/src/app/ui/components/modals/entity-picker/configs/members.tsx`
- Field: `website/src/app/ui/components/form/FormEntityPickerField.tsx`
- Docs: `docs/platforms/website/flow-form-foundation.md` §3.6
