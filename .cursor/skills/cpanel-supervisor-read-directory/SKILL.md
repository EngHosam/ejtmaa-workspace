---
name: cpanel-supervisor-read-directory
description: >-
  Ships or extends a cpanel supervisor read-only directory (ResultLane list +
  detail) inheriting DATA_ADAPTERS.SUPERVISOR_GQL, with Stats extras, PageStateLane
  Loadable/Empty/LaneFailed hosts, nested MPagesRoutes params, and no new adapter/Forms keys.
  Use when adding or changing supervisor list/detail modules under cpanel/.
---

# CPanel supervisor read directory

## When to Use

- Adding another supervisor list + read-only detail under `cpanel/` (customers is the shipped template).
- Changing customers list/detail, `Stats`, `PageStateLane`, or those routes/drawer tiles.
- Reviewing whether a new supervisor screen should be DataTable, a new `DATA_ADAPTERS` key, or home KPIs.

## Instructions

1. Read `docs/platforms/cpanel/customer-management.md` and `.cursor/rules/cpanel-supervisor-read-directory.mdc`.
2. Register `supervisorRouter` paths: list then `:id`. `MPagesRoutes` nested `params`. Thin `MyPage` files.
3. One `useShallowAdapter` per route, `inheritedAdapterIdentify: DATA_ADAPTERS.SUPERVISOR_GQL`. List: mount-private string + `listable`. Detail: no identify, no `listable`, `section` root. No new registry keys.
4. List UI: `SearchField` Enter + history key, `ResultLane` + matching skeleton, `Stats` only from query extras. Copy nested org as مؤسسة.
5. Detail UI: `FlexContainer dir="column" flx_1`. `PageStateLane` for loading (`Loadable` 3rem), `Empty`, `LaneFailed`. Do not edit `Wrong` / `Loadable` APIs. Compute overlay flags in the hook.
6. Keep `SupervisorMainLayout` content as a `Col` so `flx_1` fills under chrome. Overlay inner host is `relative` + `flx_1` inside the same padded `Col` as success content.
7. Refresh GQL mirrors by command copy. `ar.ts` only. No writes/Forms unless the task explicitly ships requesters.
8. Update C17 docs in the same task (`customer-management.md` and indexes). Verify with `cpanel` `yarn type-check`.
