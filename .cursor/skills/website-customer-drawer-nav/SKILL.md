---
name: website-customer-drawer-nav
description: >-
  Aligns CustomerDrawer tile IA and identifies with backend CustomerSchema /
  customer settings before editing drawer nav or documenting customer routes.
  Use when changing drawer items, planning customer workspace routes, or
  reviewing drawer vs backend drift.
---

# Website customer drawer nav

## When to Use

- Adding, removing, renaming, or reordering `CustomerDrawer` tiles.
- Planning `/customer/*` routes that the drawer will enable.
- Reviewing whether drawer IA drifted toward static-info or foreign-product menus.

## Instructions

1. Read `backend/src/app/gql/schemas/CustomerSchema.ts` Query roots and website customer requesters (`requesters.website.ts` → `customer`).
2. Read current tiles in `website/src/app/ui/components/customer/CustomerDrawer.tsx` and the table in `docs/platforms/website/flow-customer-shell.md` §5.3.
3. Propose or edit tiles only from that backend surface (+ explicit support/help placeholders). Prefer **message templates** naming over “notification templates”.
4. Keep route gating: `Object.prototype.hasOwnProperty.call(routes, identify)`; do not invent pages solely to enable tiles.
5. Home tile glyph is shared `HomeMark` (`tone="onPrimary"`), not `FiHome`. Other tiles may keep Feather `Icon` or a `mark` ReactNode via `DrawerGridItem` dual branch.
6. Hero workspace chip (not a role badge): `me.organization.name` when present; else drawer i18n `workspaceFallback` (**منصة اجتماع** / **Ejtmaa platform**). Chrome: `accentActionBackground` + `accentActionText`. Do not show a customer/عميل role label.
7. When a new tile identify ships a page, also run `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md` (W38) unless it is `CustomerHome` or a bottom-tab destination.
8. Update `flow-customer-shell.md` §5.2–§5.3 and `route-registry-contract.md` §5.2 in the same change when identifies/labels/hero contract change.
9. Follow `.cursor/rules/website-customer-drawer-nav-backend-alignment.mdc`.
