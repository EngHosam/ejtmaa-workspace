# Website Documentation Index

## Foundation

| Document | Scope |
|---|---|
| [`overview.md`](overview.md) | Customer portal architecture |
| [`repository-inventory.md`](repository-inventory.md) | Repo file inventory |
| [`component-structure.md`](component-structure.md) | Folder and layer ownership |
| [`ui-foundation.md`](ui-foundation.md) | Utils + theme.ts contract |
| [`brand-identity-alignment.md`](brand-identity-alignment.md) | URL rebrand + semanticColor audit outcome |
| [`data-flow-and-gql.md`](data-flow-and-gql.md) | Adapters, requesters, GQL |
| [`graphql-mirror-and-tooling.md`](graphql-mirror-and-tooling.md) | GQL mirror sync |
| [`route-registry-contract.md`](route-registry-contract.md) | Customer route registry |
| [`ssr-boot-and-startup.md`](ssr-boot-and-startup.md) | SSR boot lifecycle |
| [`shared-ui-and-shell.md`](shared-ui-and-shell.md) | Shell components |
| [`page-error.md`](page-error.md) | Error (403/404/500) page composition |
| [`landing-page.md`](landing-page.md) | Public landing page (10 sections, shell, style/color invariants) |

## Flows

| Flow | Document | Scope |
|---|---|---|
| Auth | [`flow-auth.md`](flow-auth.md) | Customer auth on `/website` |
| Form foundation | [`flow-form-foundation.md`](flow-form-foundation.md) | Shared form patterns |
| Settings | [`flow-settings.md`](flow-settings.md) | Account settings |
| Notifications | [`flow-notifications.md`](flow-notifications.md) | Notification list |
| Static info | [`flow-static-info-pages.md`](flow-static-info-pages.md) | About Ejtmaa, legal pages |
| Customer shell | [`flow-customer-shell.md`](flow-customer-shell.md) | Authed customer layout |

## Invariants

- [`../../invariants/website.md`](../../invariants/website.md)

## Change set traceability — visitor auth pages

Current-state reflection of the shipped visitor auth surface (`Login`, `Register`, `ResetPassword`). No change history.

| Path (under `website/`) | Documented where |
|---|---|
| `src/app/ui/pages/Login.tsx` | `flow-auth.md` §6.1 |
| `src/app/ui/pages/Register.tsx` | `flow-auth.md` §6.2 |
| `src/app/ui/pages/ResetPassword.tsx` | `flow-auth.md` §6.3 |
| `src/app/ui/components/auth/AuthPageShell.tsx` | `flow-auth.md` §4; `brand-identity-alignment.md` § Logo + auth shell |
| `src/app/ui/components/auth/AuthTextField.tsx` | `flow-auth.md` §5 |
| `src/app/ui/components/auth/AuthNavLink.tsx` | `flow-auth.md` §5, §7 |
| `src/app/ui/components/auth/AuthSecondaryNavButton.tsx` | `flow-auth.md` §5, §7 |
| `src/app/ui/components/form/FormTextField.tsx` | `flow-auth.md` §5.1; `flow-form-foundation.md` §2 |
| `src/app/ui/components/form/FormActionButton.tsx` | `flow-auth.md` §5.2; `brand-identity-alignment.md` § Canonical consumer pairings |
| `src/app/ui/components/form/FormInputWrapper.tsx` | `flow-auth.md` §5 (`actionArea` on password field) |
| `src/app/ui/components/Logo.tsx` (`auth` preset) | `brand-identity-alignment.md` § Logo |
| `src/app/ui/components/LandingHeader.tsx` (register CTA) | `flow-auth.md` §8 |
| `src/app/ui/layouts/LandingLayout.tsx` (drawer register nav) | `flow-auth.md` §8 |
| `src/app/services/router.ts` (`publicRoutes`, middleware) | `flow-auth.md` §2; `route-registry-contract.md` §1.1 |
| `src/resources/configs/routes.ts` | `route-registry-contract.md` §1.1; `flow-auth.md` §2 |
| `src/resources/translations/ar.ts` (`ui.pages.login/register/resetPassword`) | `flow-auth.md` §9 |
| `src/resources/translations/en.ts` (mirror) | `flow-auth.md` §9 |
| `src/types/requesters/requesters.website.ts` | `flow-auth.md` §1; `flow-form-foundation.md` §2 |
| `lib/tsconfig.tsbuildinfo` | Generated build artifact from `yarn type-check`; not narrated. |

Governance: `.cursor/rules/website-auth-flow.mdc`

## Change set traceability — brand polish

| Path (under `website/`) | Documented where |
|---|---|
| `src/resources/configs/theme.ts` (`Dims` corner radius) | `ui-foundation.md` § Corner radius tokens; `brand-identity-alignment.md`; invariant W46; `website-corner-radius-tokens.mdc` |
| `src/resources/configs/utils.ts` (`semanticDims.card.radius`) | `ui-foundation.md` § Corner radius tokens; `page-error.md` § Radius + tokens; `website-corner-radius-tokens.mdc` |
| `src/app/ui/components/Logo.tsx` (presets + sizes) | `brand-identity-alignment.md` § Logo; `shared-ui-and-shell.md` § 3/4; invariant W45; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Header.tsx` (logo no-frame) | `shared-ui-and-shell.md` § 3; `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Footer.tsx` (logo no-frame) | `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Drawer.tsx` (logo no-frame) | `shared-ui-and-shell.md` § 4; `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/pages/Error.tsx` (full page) | `page-error.md` (full composition + brand treatment); `brand-identity-alignment.md` § Canonical consumer pairings |
| `src/resources/translations/ar.ts` (`error.title` key) | `page-error.md` § Translation contract; `page-error.md` § Page title |
| `public/images/{dark,light}_logo.png` (binary brand assets) | `brand-identity-alignment.md` § Logo (asset paths + scheme swap). Binary; not narrated line-by-line. |
| `lib/tsconfig.tsbuildinfo` | Generated build artifact from `yarn type-check`; not narrated. |

`ar.ts` also carries a brand-rename pass (`مصدرية` → `اجتماع` in `app.title`, `footerTitle`, and `brand` strings), made outside this agent session; reflected as the current app title value, not narrated key-by-key.

## Change set traceability — customer baseline

Current-state reflection of the customer-baseline change set: the `website/` scaffold now targets the `CUSTOMER` + `visitor` actor model on mount `/website`, with the supervisor/cpanel surfaces removed and customer-native replacements added. No change history.

### Removed (13 deleted paths)

| Path (under `website/`) | Was | Documented where |
|---|---|---|
| `src/app/ui/pages/Customers.tsx` | Supervisor customer-list page | `route-registry-contract.md` §1.1 (shipped routes exclude it) |
| `src/app/ui/pages/Customer.tsx` | Supervisor customer-detail page | `route-registry-contract.md` §1.1 |
| `src/app/ui/components/customer/CustomersTable.tsx` | Supervisor customer table | `repository-inventory.md` (not part of shipped customer UI) |
| `src/app/ui/components/customer/CustomerStatsSection.tsx` | Supervisor stats section | `repository-inventory.md` |
| `src/app/ui/components/customer/CustomerIdentityCell.tsx` | Supervisor identity cell | `repository-inventory.md` |
| `src/app/ui/components/customer/customer.graphql` | Supervisor customer operation | `graphql-mirror-and-tooling.md` §5 (supervisor not mirrored) |
| `src/app/ui/components/customer/customers.graphql` | Supervisor customers operation | `graphql-mirror-and-tooling.md` §5 |
| `src/types/customer.ts` | `CustomerFormType` helper | `route-registry-contract.md` §1.1 (no `Customer` route) |
| `src/types/gql/definitions/base.graphql` (old) | Supervisor-contaminated base SDL | `graphql-mirror-and-tooling.md` §2 |
| `src/types/gql/definitions/supervisor.graphql` | Supervisor SDL | `graphql-mirror-and-tooling.md` §5 |
| `src/types/gql/gql-types/base.ts` (old) | Old generated types | `graphql-mirror-and-tooling.md` §2 |
| `src/types/gql/gql-types/supervisor.ts` | Supervisor generated types | `graphql-mirror-and-tooling.md` §5 |
| `src/types/requesters/requesters.cpanel.ts` | Cpanel requester map | `repository-inventory.md` § Requesters; `data-flow-and-gql.md` § Actor maps |

### Added (8 new paths)

| Path (under `website/`) | Role | Documented where |
|---|---|---|
| `src/types/requesters/requesters.website.ts` | `RequestersMap` (visitor.auth, customer.customer, customer.notification) | `repository-inventory.md` § Requesters; `data-flow-and-gql.md` § Actor maps; `overview.md` § Backend coupling |
| `src/types/gql/definitions/base.graphql` | Shared SDL (scalars, `_Ability`, `_Notification*`, interfaces) | `graphql-mirror-and-tooling.md` §2; `repository-inventory.md` § GQL mirrors |
| `src/types/gql/definitions/customer.graphql` | Customer SDL (`_Me`, `_Organization`, `_Member`, `_MessageTemplate`, `_Meeting`, `Query { me, notifications, organization, members, member, messageTemplates, messageTemplate, meetings, meeting }`) | `graphql-mirror-and-tooling.md` §2; `repository-inventory.md` § GQL mirrors |
| `src/types/gql/definitions/shared.graphql` | Shared `Query { notifications }` extension stub | `graphql-mirror-and-tooling.md` §2; `repository-inventory.md` § GQL mirrors |
| `src/types/gql/gql-types/base.ts` | Generated shared types | `graphql-mirror-and-tooling.md` §2 |
| `src/types/gql/gql-types/customer.ts` | Generated customer types | `graphql-mirror-and-tooling.md` §2; `data-flow-and-gql.md` § Read path |
| `src/app/ui/components/customer/graphql.config.yml` | Subtree tooling placeholder (no screens yet) | `graphql-mirror-and-tooling.md` §3 |
| `src/app/ui/pages/customer/graphql.config.yml` | Subtree tooling placeholder (no screens yet) | `graphql-mirror-and-tooling.md` §3 |

### Modified — behavioral

| Path (under `website/`) | Change | Documented where |
|---|---|---|
| `src/resources/configs/urls.ts` | `BASE_URL` test mode `/cpanel` → `/website` | `overview.md` § Backend coupling; invariant W44 |
| `src/app/services/auth.ts` | `AuthedAs = "CUSTOMER"`; `canDoAction` reads `customer.permissions` | `ssr-boot-and-startup.md` §4; `shared-ui-and-shell.md` §1.1 |
| `src/app/services/socket.ts` | Authed socket namespace `customer` | `data-flow-and-gql.md` § Socket |
| `src/app/services/router.ts` | `publicRoutes` includes `Register`, `ResetPassword`; `getMyHomeIdentify = "Home"` | `route-registry-contract.md` §1.1; `flow-auth.md` §2 |
| `src/app/services/global.ts` | Trailing semicolon fix only | — (trivial) |
| `src/resources/configs/axios/api.ts` | `FORMS.SUPERVISOR.R` → `FORMS.CUSTOMER.R` (`/forms/customer/requester/...`); imports `requesters.website` | `data-flow-and-gql.md` § Write path table |
| `src/resources/configs/routes.ts` | Removed `Customers`/`Customer` routes + `MPagesRoutes` entries | `route-registry-contract.md` §1.1 |
| `src/resources/configs/store/reduces/auth.ts` | `supervisor` → `customer` slice | `shared-ui-and-shell.md` §1.1; `ssr-boot-and-startup.md` §4 |
| `src/resources/configs/store/data-adapters.ts` | `//supervisor` → `//customer` comment on `ADAPTER1` | `data-flow-and-gql.md` § Adapter enterMode |
| `src/resources/configs/socket/events.ts` | Supervisor events → single `onCustomerEventDate` | `data-flow-and-gql.md` § Socket |
| `src/types/events.ts` | Supervisor payloads → `OnCustomerEventDate { type: "UPDATED" }` | `data-flow-and-gql.md` § Socket |
| `src/app/ui/pages/Login.tsx` | Email+password login; `sub: "login"`; split `AuthPageShell` | `flow-auth.md` §6.1 |
| `src/app/ui/pages/Error.tsx` | Consolidated `useRouter` + `useCurrentParams` import | `page-error.md` |
| `src/app/ui/layouts/main-layout/drawer.ts` | Drawer → `business` section with `dashboard` + `logout` only | `shared-ui-and-shell.md` §1.1 |
| `src/resources/translations/ar.ts` | Removed cpanel nav labels; `owner` → `Customer` | `repository-inventory.md`; invariant W48 |
| `src/resources/translations/en.ts` | Mirror of `ar.ts` removals; `owner` → `Customer` | `repository-inventory.md`; invariant W48 |
| `graphql.config.yml` (root) | Schema ref `supervisor.graphql` → `customer.graphql` | `graphql-mirror-and-tooling.md` §3 |
| `src/app/ui/components/graphql.config.yml` | Schema ref → `base.graphql` only | `graphql-mirror-and-tooling.md` §3 |
| `src/app/ui/pages/graphql.config.yml` | Schema ref → `base.graphql` only | `graphql-mirror-and-tooling.md` §3 |

### Modified — formatting only (no runtime behavior change)

| Path (under `website/`) | Change |
|---|---|
| `src/app/ui/components/DataTable.tsx` | Destructuring param line alignment |
| `src/app/ui/components/Drawer.tsx` | Destructuring param line alignment |
| `src/app/ui/components/Footer.tsx` | Destructuring param line alignment |
| `src/app/ui/components/Header.tsx` | Destructuring param line alignment + import order |
| `src/app/ui/components/LanguageSwitch.tsx` | Destructuring param line alignment |
| `src/app/ui/components/Loadable.tsx` | Destructuring param line alignment |
| `src/app/ui/components/Logo.tsx` | Type literal spacing |
| `src/app/ui/components/ThemeModeSwitch.tsx` | Destructuring param line alignment |
| `src/app/ui/components/auth/AuthTextField.tsx` | Thin wrapper; passes `type` including `tel` |
| `src/app/ui/components/form/FormTextField.tsx` | `readOnly` during injection; optional `actionArea`; `placeholder` default `""` |
| `src/app/ui/components/form/FormActionButton.tsx` | `tone: "secondary"` for cross-auth CTAs |
| `src/app/ui/pages/UiMockup.tsx` | Destructuring param line alignment + import consolidation |
| `src/resources/configs/theme.ts` | `FontFamilies` single-quote → escaped-double-quote strings (no value change) |
| `lib/tsconfig.tsbuildinfo` | Generated build artifact from `yarn type-check`; not narrated. |

## Governance

- `.cursor/rules/website-platform-governance.mdc`
- `.cursor/rules/website-customer-only-import-boundary.mdc`
- `.cursor/skills/website-platform-governance/SKILL.md`
