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

1. Add `modals/entity-picker/configs/<ident>.tsx` with `gql`, `listable`, `buildFilter(search, extras?)`, optional `searchable` (default true), `Card`, `getValue` / `getLabel` / optional `getAvatarUrl`. Register in `configs/index.ts`. Types: `entity-picker/types.ts`.
2. Prefer the **same page Card** inside `SelectableEntityCard`. **Always forward `selected` into the Card** when the page card supports selection chrome (members + message channels). Do not drop `selected` for a new ident.
3. Modal (`EntityPickerModal`): render `SearchField` only when `searchable` (prop or config). Enter-committed text search when searchable; list Col `minH` when empty/busy so `Empty` overlay is visible; `LoadMoreButton` when `thereMoreRecords` — `.cursor/rules/website-custom-scroll-contract.mdc`.
4. Field (`FormEntityPickerField`): single value `{ value, label, avatarUrl? }`; multi = array; optional `filter` extras passed into `buildFilter`. Edit-form `read` must hydrate that shape via related `model.forSelect(lang)` — `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`.
5. Register modal in `resources/configs/store/modals.ts` as `ENTITY_PICKER` if not already present. i18n under `ui.modals.entityPicker.*`.
6. Update `docs/platforms/website/flow-form-foundation.md` §3.6 (+ consumer flow doc). Verify `yarn type-check` in `website/`.

## Canonical reference

- Modal: `website/src/app/ui/components/modals/EntityPickerModal.tsx`
- Config: `website/src/app/ui/components/modals/entity-picker/configs/members.tsx`
- Config (no search + `status: ACTIVE` + optional type + **selected**): `…/configs/messageChannels.tsx`
- Field: `website/src/app/ui/components/form/FormEntityPickerField.tsx`
- Docs: `docs/platforms/website/flow-form-foundation.md` §3.6
- Backend read hydrate: `docs/platforms/backend/patterns/requester-read-select-hydrate.md`