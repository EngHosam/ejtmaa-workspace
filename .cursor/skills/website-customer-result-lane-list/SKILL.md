---
name: website-customer-result-lane-list
description: >-
  Ships an authenticated customer ResultLane directory with CUSTOMER_GQL inherit,
  history-backed Enter search, load-more, and mirrored GQL filter. Use when adding
  or extending website customer list screens like members (not DataTable).
---

# Website customer ResultLane list

## When to Use

- Adding or changing a customer workspace **card-grid directory** (ResultLane + load-more), not a DataTable.
- Wiring **server-side** list search/filter through route history + GQL `filter`.
- Extending the members pattern to another org-owned customer list.

## Instructions

1. Confirm backend SDL exposes the list root + filter input, schema passes `{ me: true, filter }` (or documented parent), and the entity bridge maps filter in `getOrmFindOptions` only when it adds real policy. Follow `.cursor/rules/gql-root-parent-payload-contract.mdc` and the domain contract under `docs/platforms/backend/contracts/`.
2. Mirror `customer.graphql` + `gql-types/customer.ts` into `website/src/types/gql/` by command-copy (no hand-edit). Run backend `yarn generate-types` / `yarn type-check` when the SDL changed.
3. Register or reuse `DATA_ADAPTERS.CUSTOMER_GQL` → `API.DATA_ADAPTERS.CUSTOMER.GQL`. Use a **mount-private** adapter id (e.g. `"customer-members"`) with `inheritedAdapterIdentify: CUSTOMER_GQL` and `listable` matching the GQL root field. One adapter per route (W36 / `website-one-adapter-per-route.mdc`).
4. Put data ownership in `components/customer/hooks/useCustomer*.ts`:
   - `useWithHistoryState({ key })` for committed query
   - draft search + Enter commit (`SearchField` `onSubmit`)
   - `mLoad({ reload: true, query })` on committed query change; same `query` on load-more/refresh
   - Never client-filter loaded rows — `.cursor/rules/website-customer-list-history-search.mdc`
5. Keep the page thin (`MyPage` → screen). Screen composes `SectionHeading`, search, `ResultLane`, and a product card. Chrome-only Add/Edit is allowed **without** inventing requesters/routes.
6. Reuse shared `ResultLane` / `SearchField` / `LoadMoreButton` / `Wrong` / `SectionHeading`. Keep skeleton shape aligned with the card — `.cursor/rules/website-result-lane-skeleton-shape.mdc`.
7. i18n ar/en for page + wrong overlays. Helmet title on the screen.
8. Document under `docs/platforms/website/flow-*.md` (extend or add) and refresh indexes (`README.md`, `overview.md`, `data-flow-and-gql.md`, route registry). Update the backend domain contract when filter ships.
9. Verify with existing scripts only: website `yarn type-check`; backend `yarn generate-types` + `yarn type-check` when contracts changed. Style props: website Utils / W29.

## Canonical reference

- Hook: `website/src/app/ui/components/customer/hooks/useCustomerMembers.ts`
- Flow: `docs/platforms/website/flow-customer-members.md`
- Backend: `docs/platforms/backend/contracts/member-domain.md`
- Subpage shell/breadcrumb: `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
