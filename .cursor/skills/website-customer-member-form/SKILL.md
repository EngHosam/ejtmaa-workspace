---
name: website-customer-member-form
description: >-
  Ships or extends the website CustomerMemberForm (multi-path create/update,
  Forms.CUSTOMER_MEMBER, FormAvatarField, list Add/Edit wiring). Use when changing
  member write UI, member form routes, or avatar upload on the customer portal.
---

# Website customer member form

## When to Use

- Adding fields, actions, or routes for member create/edit/delete on `website/`.
- Wiring list Add/Edit to the form, or changing avatar upload behavior.
- Aligning a new customer create+update form with the members pattern.

## Instructions

1. Confirm backend `MemberRequester` + `Customer.Ability.MEMBER` + `requesters.website` map already match the intended subs (`read` | `create` | `update` | `delete`). Read `docs/platforms/backend/contracts/member-domain.md` §9. Do not invent cascade delete; roster/chairperson blocks stay server-side.
2. Keep one multi-path identify `CustomerMemberForm` with `path: { create, update }` and param-aware breadcrumb — `.cursor/rules/website-multi-path-form-routes.mdc`. Register after `CustomerMembers`. Mirror `MPagesRoutes: { id?: string }`.
3. Register `Forms.CUSTOMER_MEMBER` → `API.FORMS.CUSTOMER.R("member")`. Mirror `customer.member` in `website/src/types/requesters/requesters.website.ts`.
4. Href builders only via `resources/configs/customer/formRoute.ts` (`buildCustomerMemberFormHref`). Wire list Add/Edit with `nav.push(...)`.
5. Screen pattern (`CustomerMemberFormScreen`):
   - `isUpdate = !!id` from `useCurrentParams`
   - create: stable `formIdentify` + `removeOnExit: false`; update: `values: { member: id }`, `didEntered` → `send({ sub: "read" })`, `Loadable` while read in flight
   - Header row: `SectionHeading` with `onBack`/`backLabel` + primary save (+ neutral delete on update). Actions are **not** full-width under the fields.
   - Fields: `FormAvatarField` + name / email / mobile. Email/mobile may use meeting-link `subTitle`. No placeholders unless product asks.
   - Submit/delete: `submittingRef`; `afterSentSuccess: nav.back()`; no manual success toast — `.cursor/rules/website-form-success-toast-automatic.mdc`
6. Avatar: `.cursor/rules/website-form-avatar-field.mdc`. Text inputs: `.cursor/rules/website-form-text-field-direction.mdc`. Utils styles: W29 / website-utils-style-prop-audit.
7. i18n ar/en under `ui.pages.customer.memberForm.*`. Update `flow-customer-members.md`, `flow-form-foundation.md`, and `route-registry-contract.md` in the same task.
8. Verify: `yarn type-check` in `website/` (and `backend/` if requester/ability changed).

## Canonical reference

- Screen: `website/src/app/ui/components/customer/members/CustomerMemberFormScreen.tsx`
- Flow: `docs/platforms/website/flow-customer-members.md` §5
- Backend: `docs/platforms/backend/contracts/member-domain.md` §9
- Sibling list skill: `.cursor/skills/website-customer-result-lane-list/SKILL.md`
