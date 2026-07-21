---
name: website-customer-form-modal
description: >-
  Ships customer-only registered form modals under customer/modals (ConfirmModal
  pattern + FormModal chrome). Use when adding or fixing meeting/details write
  modals or other customer form modals on website/.
---

# Website customer form modal

## When to Use

- Adding or changing a customer-only write modal (meeting basics, participant add, agenda/decision subject, or a new sibling).
- Replacing a generic `openForm({ render })` / Redux JSX-body anti-pattern.
- Reviewing modal file placement vs shared `components/modals/`.

## Instructions

1. Place the Component under `website/src/app/ui/components/customer/modals/`. Shared chrome/pickers stay in `components/modals/` — see `.cursor/rules/website-customer-form-modal-placement.mdc`.
2. Register a dedicated identity in `resources/configs/store/modals.ts` (`Component` + `cancelable: true` + `MModalsMap` props type).
3. Export `openX(props)` that calls `openModal(myInstance)(IDENT)({ ...props })` with **data + callbacks only** (no `render` factory).
4. Compose presentational `FormModal` for title/subtitle chrome; body owns `useShallowForm` + `FormProvider` + `Form*` fields + `submittingRef` + truthful `d.send({ sub })`.
5. Do not share the parent page form into the modal. Do not edit `ModalBase`. No manual success toast (W11 / form success toast rule).
6. Wire the screen with the open helper. Update `flow-customer-meetings.md` / `flow-form-foundation.md` in the same task.
7. Verify: `yarn type-check` in `website/`.

## Canonical reference

- `MeetingBasicsModal.tsx` / `openMeetingBasics`
- `MeetingParticipantAddModal.tsx` / `openMeetingParticipantAdd`
- `MeetingSubjectModal.tsx` / `openMeetingSubject`
- Chrome: `components/modals/FormModal.tsx`
- Sibling shared pattern: `ConfirmModal` / `confirm()`
- Flow: `docs/platforms/website/flow-customer-meetings.md` §6.3–§6.4
