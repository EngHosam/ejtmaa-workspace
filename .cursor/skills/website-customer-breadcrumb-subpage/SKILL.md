---
name: website-customer-breadcrumb-subpage
description: >-
  Ships an authed CUSTOMER_MAIN drawer/header subpage with route breadcrumb,
  fixed CustomerSubHeader, empty or list page shell, and docs. Use when adding
  customer workspace routes under /customer/* that are not the role home.
---

# Website customer breadcrumb subpage

## When to Use

- Adding a new `mustAuthedAs: ["CUSTOMER"]` route on `CUSTOMER_MAIN` reached from the drawer or header (not bottom-tab home).
- Wiring breadcrumb meta, `CustomerSubHeader` visibility, or `HomeMark` consumers.
- Reviewing W38 compliance for a customer subpage.

## Instructions

1. Read `.cursor/rules/website-authed-drawer-subpage-governance.mdc`, `.cursor/rules/website-breadcrumb-product-placement.mdc`, and `docs/platforms/website/flow-customer-shell.md` §7.1.
2. Register the route in `website/src/resources/configs/routes.ts` (customer section + matching `MPagesRoutes` member) via `customerRouter(...)`, `layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`, and `breadcrumb: { parent: "CustomerHome", label: tr => tr.ui.pages.customer.<feature>.title }` unless a deeper parent is documented.
3. Add a thin `MyPage` under `website/src/app/ui/pages/customer/`. Empty `Main()` is valid until the feature body ships. List screens later add `SectionHeading` + `pt={2}` per authed-drawer governance. For ResultLane + history search directories, follow `.cursor/skills/website-customer-result-lane-list/SKILL.md`.
4. Confirm `CustomerMainLayout` already toggles `CustomerSubHeader` from `routes[identify]?.breadcrumb` — do **not** hardcode identify lists.
5. Keep breadcrumb product files under `ui/components/` (`Breadcrumb`, `useBreadcrumbs`, `HomeMark`, `CustomerSubHeader`). Never place them in `ui/base/`.
6. If the page is a new drawer tile identify, follow `.cursor/skills/website-customer-drawer-nav/SKILL.md` for IA alignment; home tile continues to use `HomeMark` `tone="onPrimary"`.
7. Add i18n keys (ar/en mirrored): page title under `ui.pages.customer.*` and any drawer label under `ui.layouts.customerMainLayout.drawer.*`.
8. Update `route-registry-contract.md` §5.2 and the feature / shell flow doc in the same task.
9. Run `yarn type-check` in `website/`. Style props must follow W29 / `website-utils-style-prop-audit`.

## Related

- `.cursor/rules/website-breadcrumb-descendant-root-label.mdc` (W35)
- `.cursor/rules/website-customer-utils-composed-marks.mdc` (`HomeMark`)
- `.cursor/rules/website-route-reactive-components.mdc` (W24)
- `docs/invariants/website.md` W38
