---
name: website-customer-organization-form
description: >-
  Ships or extends the website CustomerOrganization settings form
  (Forms.CUSTOMER_ORGANIZATION, read+upsert, FormAvatarField logo_file,
  FormColorField). Use when changing organization write UI or branding fields
  on the customer portal.
---

# Website customer organization form

## When to Use

- Adding or changing fields/actions on `/customer/organization`.
- Wiring organization requester subs (`read` | `upsert`) to the form.
- Reusing `FormAvatarField` / `FormColorField` for org branding.

## Instructions

1. Confirm backend `OrganizationRequester` + `Customer.Ability.ORGANIZATION` + `requesters.website` map match (`read` | `upsert`). Read `docs/platforms/backend/contracts/organization-domain.md` §9. Do not accept `custom_domain` or `status` from the form; create forces `ACTIVE` server-side.
2. Keep a **single-path** identify `CustomerOrganization` → `customerRouter("/organization")` with breadcrumb `{ parent: "CustomerHome", label: tr => tr.ui.pages.customer.organization.title }`. Mirror empty `MPagesRoutes` member. Not multi-path.
3. Register `Forms.CUSTOMER_ORGANIZATION` → `API.FORMS.CUSTOMER.R("organization")`. Mirror `customer.organization` in `website/src/types/requesters/requesters.website.ts` (W18).
4. Screen pattern (`CustomerOrganizationScreen`):
   - `didEntered` → `send({ sub: "read" })` always (empty defaults when no org).
   - Header: `SectionHeading` **without** `onBack` + primary Save (`sub: "upsert"`).
   - `afterSentSuccess`: re-`read` and **stay on page** (not `nav.back()`).
   - `submittingRef`; no manual success toast — `.cursor/rules/website-form-success-toast-automatic.mdc`.
   - Fields: `FormAvatarField` `name="logo_file"`; name / description / subdomain; `FormColorField` with `BrandColors` fallbacks.
5. Avatar/logo: `.cursor/rules/website-form-avatar-field.mdc`. Colors: `.cursor/rules/website-form-color-field.mdc`. Text: `.cursor/rules/website-form-text-field-direction.mdc`. Utils: W29 / website-utils-style-prop-audit.
6. Subdomain is **required** server-side (min 4, English alpha, BadWords/reserved list, uniqueness). Surface Joi keys via automatic form errors — do not invent client-only validation frameworks.
7. i18n ar/en under `ui.pages.customer.organization.*`. Update `flow-customer-organization.md`, `flow-form-foundation.md`, and `route-registry-contract.md` in the same task.
8. Verify: `yarn type-check` in `website/` (and `backend/` if requester/ability changed).

## Canonical reference

- Screen: `website/src/app/ui/components/customer/organization/CustomerOrganizationScreen.tsx`
- Flow: `docs/platforms/website/flow-customer-organization.md`
- Backend: `docs/platforms/backend/contracts/organization-domain.md` §9
- Sibling form skill: `.cursor/skills/website-customer-member-form/SKILL.md`
