---
name: website-customer-organization-form
description: >-
  Ships or extends the website CustomerOrganization settings form
  (Forms.CUSTOMER_ORGANIZATION, read+upsert, FormAvatarField logo_file,
  FormColorField, setup gate, post-save redirect home). Use when changing
  organization write UI, branding fields, or org onboarding gate on the
  customer portal.
---

# Website customer organization form

## When to Use

- Adding or changing fields/actions on `/customer/organization`.
- Wiring organization requester subs (`read` | `upsert`) to the form.
- Reusing `FormAvatarField` / `FormColorField` for org branding.
- Changing incomplete-org setup gate or post-save navigation.

## Instructions

1. Confirm backend `OrganizationRequester` + `Customer.Ability.ORGANIZATION` + `requesters.website` map match (`read` | `upsert`). Read `docs/platforms/backend/contracts/organization-domain.md` §9. Do not accept `custom_domain` or `status` from the form; create forces `ACTIVE` server-side.
2. Customer GQL org for the portal is **`_Me.organization` only** — no customer root `Query.organization`. Keep `OrganizationBridge.GetOneParent` as model nests (`CustomerModel` | Member | MessageTemplate | Meeting). Mirror after SDL changes.
3. Keep a **single-path** identify `CustomerOrganization` → `customerRouter("/organization")` with breadcrumb `{ parent: "CustomerHome", label: tr => tr.ui.pages.customer.organization.title }`. Mirror empty `MPagesRoutes` member. Not multi-path.
4. Register `Forms.CUSTOMER_ORGANIZATION` → `API.FORMS.CUSTOMER.R("organization")`. Mirror `customer.organization` in `website/src/types/requesters/requesters.website.ts` (W18).
5. Screen pattern (`CustomerOrganizationScreen`):
   - `didEntered` → `send({ sub: "read" })` always (empty defaults when no org).
   - When `useMe` org is incomplete: show `setupRequiredAlert` status banner above the header.
   - Header: `SectionHeading` **without** `onBack` + primary Save (`sub: "upsert"`).
   - `afterSentSuccess`: `useRouter().redirect({ identify: "CustomerHome", replace: true })` — not form re-`read`, not `nav.push`.
   - `submittingRef`; no manual success toast — `.cursor/rules/website-form-success-toast-automatic.mdc`.
   - Fields: `FormAvatarField` `name="logo_file"`; name / description / subdomain with FormTextField `suffix` from `ORG_PUBLIC_DOMAIN` (`ejtmaa.live`); `FormColorField` with `BrandColors` fallbacks.
6. Setup gate: `.cursor/rules/website-customer-organization-setup-gate.mdc` — incomplete `me.organization` (no id or no subdomain) forces `CustomerOrganization` via `applyRouterMiddleware`. Ensure `useMe` `coreQuery` selects gate fields (`organization { id … subdomain }`).
7. Avatar/logo: `.cursor/rules/website-form-avatar-field.mdc`. Colors: `.cursor/rules/website-form-color-field.mdc`. Text + suffix LTR row: `.cursor/rules/website-form-text-field-direction.mdc`. Utils: W29 / website-utils-style-prop-audit.
8. Subdomain is **required** server-side (min 4, English alpha, BadWords/reserved list, uniqueness). Surface Joi keys via automatic form errors — do not invent client-only validation frameworks.
9. i18n ar/en under `ui.pages.customer.organization.*` (include `setupRequiredAlert`). Update `flow-customer-organization.md`, `flow-auth.md` §2.1, `flow-form-foundation.md`, and `route-registry-contract.md` in the same task when contracts change.
10. Verify: `yarn type-check` in `website/` (and `backend/` if requester/ability/GQL changed).

## Canonical reference

- Screen: `website/src/app/ui/components/customer/organization/CustomerOrganizationScreen.tsx`
- Gate: `website/src/app/services/router.ts`
- Me hydrate: `website/src/app/ui/components/customer/hooks/useMe.tsx`
- Flow: `docs/platforms/website/flow-customer-organization.md`
- Backend: `docs/platforms/backend/contracts/organization-domain.md` §5 / §9
- Sibling form skill: `.cursor/skills/website-customer-member-form/SKILL.md`
