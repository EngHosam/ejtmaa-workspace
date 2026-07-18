---
name: website-platform-governance
description: >-
  Enforces Ejtmaa website platform governance for work under `website/`. Use when
  creating, modifying, fixing, reviewing, or planning the SSR Customer frontend,
  especially routes, pages, layouts, shared UI, `ui/base`, `resources/configs`,
  adapters, forms, auth, GQL wiring, or web-native feature work.
---

# Ejtmaa website platform governance

## When to Use

- Any task under `website/` that touches routes, layouts, UI, adapters, forms, auth, or GQL.
- Reviewing website changes for compliance with documented foundation and invariants.

## Read first

- `.cursor/rules/website-platform-governance.mdc`
- `docs/platforms/website/overview.md`
- `docs/platforms/website/repository-inventory.md`
- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/ui-foundation.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/website/shared-ui-and-shell.md`
- `docs/platforms/website/ssr-boot-and-startup.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `docs/platforms/website/flow-auth.md`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/flow-settings.md`
- `docs/platforms/website/flow-notifications.md`
- `docs/platforms/website/flow-static-info-pages.md`
- `docs/platforms/website/flow-customer-shell.md`
- `.cursor/rules/website-route-registry-governance.mdc`
- `.cursor/rules/website-route-static-before-parametric.mdc`
- `.cursor/rules/website-customer-drawer-nav-backend-alignment.mdc`
- `.cursor/rules/website-customer-utils-composed-marks.mdc`
- `docs/invariants/website.md`

## Instructions

1. Confirm scope: visitor + customer actors on `/website`; GQL mirrors `base` + `customer` only.
2. Route changes must follow `route-registry-contract.md` (section blocks, `customerRouter`, static-before-parametric).
3. UI work uses `Utils.tsx` + `theme.ts` + `utils.ts`; product UI in `ui/components/`, not `ui/base/`.
4. Authed customer adapter reads use `API.DATA_ADAPTERS.CUSTOMER.GQL`; visitor/global reads use `DATA_ADAPTERS.GQL`. Writes via `API.FORMS.R` (visitor) or `API.FORMS.CUSTOMER.R`.
5. Drawer/header-launched authed subpages declare route `breadcrumb` (W38).
6. Customer drawer IA: check `CustomerSchema` / `.cursor/skills/website-customer-drawer-nav/SKILL.md` before editing tiles.
7. After material changes, update matching website docs and cited invariants in the same task.
8. Run `yarn type-check` in `website/` when touching types, routes, or GQL mirrors.

## Compliance checklist

- [ ] Routes grouped public → customer → base; `Error` last.
- [ ] Customer paths use `customerRouter(...)`; fixed segments before `:id` on same prefix.
- [ ] `MPagesRoutes` mirrors `routes` section blocks.
- [ ] GQL mirrors limited to `base` + `customer` under `src/types/gql/**`.
- [ ] Requester maps in `types/requesters/requesters.website.ts` match backend website surface.
- [ ] Brand colors from `theme.ts` (`#0B2057`, `#EC6901`); no parallel styling system.
- [ ] Auth routing middleware stays in `MyPage.tsx`, not `web-core.router.beforePageLoading`.
- [ ] Locale switch via `changeLocale` + full reload only.

## Scope

- **Customer + visitor** on `/website`.
- Brand/theme source of truth: `website/src/resources/configs/theme.ts` (navy `#0B2057`, orange `#EC6901`).
- UI foundation: `ui/base/components/Utils.tsx` + `theme.ts` + `utils.ts`.
