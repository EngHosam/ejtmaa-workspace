---
name: website-customer-meeting-form
description: >-
  Ships or extends the website customer meeting create form and meetings
  directory (Forms.CUSTOMER_MEETING, FormChoice/DateTime/EntityPicker, list
  filters). Use when changing meeting create UI, list filters, or meeting routes.
---

# Website customer meeting form + directory

## When to Use

- Changing meeting create fields, success navigation, or chairperson picker wiring.
- Extending the meetings list (filters, card, skeleton) on `website/`.
- Aligning a new create-only customer form with the meetings pattern.

## Instructions

1. Confirm backend `MeetingRequester.create` + `Customer.Ability.MEETING` + `requesters.website` `customer.meeting: "create"` — `docs/platforms/backend/contracts/meeting-domain.md` §9. Do not invent update/delete or plan quota in this slice.
2. Routes: `CustomerMeetings` → `CustomerMeetingForm` (`/meetings/form`) **before** `CustomerMeetingDetails` (`/meetings/:id`) — W41. Href: `buildCustomerMeetingFormHref` only.
3. Register `Forms.CUSTOMER_MEETING` → `API.FORMS.CUSTOMER.R("meeting")`. Mirror requester map on website.
4. Form screen: create-only stable identify + `removeOnExit: false`; `submittingRef`; fields subject / type (`FormChoiceField`) / datetime (`FormDateTimeField`) / min_members / chairperson (`FormEntityPickerField` `ident="members"`); `afterSentSuccess` → details with `replace: true`; no manual success toast.
5. List: `useCustomerMeetings` history key `meetings`; Enter search + status chips with `{ reset: true }` when clearing; `ResultLane` + `MeetingCardSkeleton`; **no Edit** on cards; W42 labels from screen into `CustomerMeetingCard`.
6. Shared controls: entity picker skill `.cursor/skills/website-entity-picker/SKILL.md`; datetime theme `.cursor/rules/website-third-party-widget-emotion-theme.mdc`.
7. i18n `ui.pages.customer.meetings*` / `meetingForm` / `meetingDetails`. Update `flow-customer-meetings.md` + `flow-form-foundation.md` + `route-registry-contract.md` in the same task.
8. Details write modals: `.cursor/skills/website-customer-form-modal/SKILL.md` + `.cursor/rules/website-customer-form-modal-placement.mdc` (`customer/modals/`, not shared `modals/` or `meetings/`).
9. Verify: `yarn type-check` in `website/` (and `backend/` if requester/SDL changed).

## Canonical reference

- Form: `website/src/app/ui/components/customer/meetings/CustomerMeetingFormScreen.tsx`
- List: `website/src/app/ui/components/customer/meetings/CustomerMeetingsScreen.tsx`
- Details modals: `website/src/app/ui/components/customer/modals/Meeting*.tsx`
- Flow: `docs/platforms/website/flow-customer-meetings.md`
- Backend: `docs/platforms/backend/contracts/meeting-domain.md` §9
- Sibling list skill: `.cursor/skills/website-customer-result-lane-list/SKILL.md`
