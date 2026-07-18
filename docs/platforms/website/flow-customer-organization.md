# Website Customer Organization (Settings Form)

Authenticated customer organization profile upsert on `CUSTOMER_MAIN`. Shell/breadcrumb contract: `flow-customer-shell.md` §7.1. Backend write contract: `docs/platforms/backend/contracts/organization-domain.md` §9. Form foundation: `flow-form-foundation.md`.

## 1) Scope

Shipped:

- Single-path route `CustomerOrganization` → `/customer/organization`.
- `Forms.CUSTOMER_ORGANIZATION` → `OrganizationRequester` (`read` | `upsert`).
- Settings-like screen: hydrate on enter, save stays on page and re-`read`s.
- Fields: logo (`logo_file`), name, description, subdomain, primary/secondary color.
- Drawer tile `CustomerOrganization` enabled via route membership.

Out of scope:

- `custom_domain` UI/API acceptance.
- Status UI (create forces `ACTIVE` server-side only).
- `SectionHeading` back control on this page (workspace settings entry, not a list child).
- Color hex schema invented on the backend (optional nullable strings).
- DNS / subdomain provisioning.

## 2) Route + breadcrumb

| Concern | Value |
|---|---|
| Identify | `CustomerOrganization` |
| Path | `customerRouter("/organization")` → `/customer/organization` |
| Layout | `CUSTOMER_MAIN` |
| Auth | `mustAuthedAs: ["CUSTOMER"]` |
| Breadcrumb | `{ parent: "CustomerHome", label: tr => tr.ui.pages.customer.organization.title }` |
| Page | thin `MyPage` → `CustomerOrganizationScreen` |
| `MPagesRoutes` | `CustomerOrganization: {}` |

Registry: `website/src/resources/configs/routes.ts`. Contract index: `route-registry-contract.md` §5.2.

## 3) Form identity

| Concern | Value |
|---|---|
| Form identity | `Forms.CUSTOMER_ORGANIZATION` → `api: API.FORMS.CUSTOMER.R("organization")` |
| Requester map | `customer.organization: "read" \| "upsert"` (W18 mirror) |
| Screen | `useShallowForm` + `inheritedFormIdentify: Forms.CUSTOMER_ORGANIZATION` |
| Enter | `didEntered` → `send({ sub: "read" })` (always; empty defaults when no org) |
| Loading | `Loadable` while `!exist` or `INIT` / (`SENDING` && `currentSub === "read"`) |
| Success | automatic toast (W11); `afterSentSuccess` → re-`send({ sub: "read" })` — **stay on page** |

Governance: `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`, `.cursor/rules/website-form-success-toast-automatic.mdc`.

## 4) Screen chrome

File: `website/src/app/ui/components/customer/organization/CustomerOrganizationScreen.tsx`

- Header row: `SectionHeading` (title + subtitle only — **no** `onBack`) + primary Save (`sub: "upsert"`).
- Actions are header-row siblings (`jc_sb`), not full-width under fields.
- Guard submit with `submittingRef`.
- Fields column `maxW={32}`:
  - `FormAvatarField` `name="logo_file"` (+ logo i18n labels),
  - `FormTextField` name / description / subdomain (subdomain has English-letters subtitle),
  - `FormColorField` primary / secondary with `fallback={BrandColors.navy|orange}`.
- No placeholders unless product asks.
- No status chips / custom domain fields.

## 5) Shared field contracts (touched this slice)

| Surface | Contract |
|---|---|
| `FormAvatarField` | Optional `name` (default `"avatar_file"`); org uses `logo_file`. Image fills circle via absolute + `cover`. Rule: `.cursor/rules/website-form-avatar-field.mdc` |
| `FormColorField` | Native `type="color"` swatch + hex text; `FormInputWrapper`; W29 (`inputReset` in `baseCssStyle` only). Rule: `.cursor/rules/website-form-color-field.mdc` |
| `FormTextField` | Inherit page direction; subdomain may use `subTitle`. |
| `IdentityAvatar` | Same absolute + `cover` fill as avatar field (header/drawer/member cards). |

## 6) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.organization.*` | title, subtitle, logo labels, field labels, subdomain subtitle, save |
| `ui.layouts.customerMainLayout.drawer.itemOrganization` | Drawer tile label (pre-existing) |

Files: `website/src/resources/translations/ar.ts`, `en.ts`.

## 7) Backend coupling

| Concern | Doc |
|---|---|
| Ability + requester + subdomain rules | `organization-domain.md` §9 |
| GQL read of org (separate from form) | `organization-domain.md` §5; `graphql-and-types.md` |

Form write path does not use a GQL data adapter; it uses customer form requesters only.

## 8) Traceability map

| Path | Role | Section |
|---|---|---|
| `website/src/resources/configs/routes.ts` | Route + `MPagesRoutes` | §2 |
| `website/src/resources/configs/store/forms.ts` | `CUSTOMER_ORGANIZATION` | §3 |
| `website/src/types/requesters/requesters.website.ts` | `customer.organization` | §3 |
| `website/src/app/ui/pages/customer/CustomerOrganization.tsx` | Thin page | §2 |
| `website/src/app/ui/components/customer/organization/CustomerOrganizationScreen.tsx` | Screen | §3–§4 |
| `website/src/app/ui/components/form/FormAvatarField.tsx` | `name` + fill | §5 |
| `website/src/app/ui/components/form/FormColorField.tsx` | Color field | §5 |
| `website/src/app/ui/components/customer/IdentityAvatar.tsx` | Avatar fill parity | §5 |
| `website/src/resources/translations/ar.ts` / `en.ts` | i18n | §6 |
| `website/src/app/ui/components/customer/CustomerDrawer.tsx` | Tile identify (pre-existing wiring) | §1 |
| `backend/src/app/orchestrator/requesters/OrganizationRequester.ts` | Write contract | §7 |
| `docs/platforms/backend/contracts/organization-domain.md` §9 | Ability + validation | §7 |

### Adjacent cleanup (same website change set)

| Path | Role | Notes |
|---|---|---|
| `website/src/app/ui/components/customer/members/CustomerMemberFormScreen.tsx` | `didEntered` uses `{d}` (no rename alias) | Trivial consistency; behavior unchanged — still `flow-customer-members.md` §5 |
| `website/lib/tsconfig.tsbuildinfo` | Build cache | Generated noise — excluded from narrative |

## 9) Verification

- `yarn type-check` in `website/`
- `yarn type-check` in `backend/` when requester/ability changes

## Related

- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/website/flow-customer-members.md` (sibling form pattern; this page differs: no back, stay + re-read)
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `.cursor/skills/website-customer-organization-form/SKILL.md`
