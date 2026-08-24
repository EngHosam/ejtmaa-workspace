---
name: cpanel-supervisor-read-directory
description: >-
  Ships or extends a cpanel supervisor ResultLane directory inheriting
  DATA_ADAPTERS.SUPERVISOR_GQL, with Stats extras, PageStateLane hosts, nested
  MPagesRoutes params, and no new DATA_ADAPTERS keys (Forms keys only when a
  requester ships). Use when adding or changing supervisor list/detail modules
  under cpanel/.
---

# CPanel supervisor read directory

## When to Use

- Adding another supervisor list + read-only detail under `cpanel/` (customers is the shipped template).
- Changing customers list/detail, `Stats`, `PageStateLane`, or those routes/drawer tiles.
- Reviewing whether a new supervisor screen should be DataTable, a new `DATA_ADAPTERS` key, a Forms write, or home KPIs (`cpanel-supervisor-home`).

## Instructions

1. Read `docs/platforms/cpanel/customer-management.md` and `.cursor/rules/cpanel-supervisor-read-directory.mdc`. Shipped templates: customers, meetings, subscriptions (read-only); plans (list + requester form).
2. Register `supervisorRouter` paths: list then `:id` (form create path before `form/:id`). `MPagesRoutes` nested `params`. Thin `MyPage` files.
3. One `useShallowAdapter` per read route, `inheritedAdapterIdentify: DATA_ADAPTERS.SUPERVISOR_GQL`. List: mount-private string + `listable`. Detail: no identify, no `listable`, `section` root. No new `DATA_ADAPTERS` keys.
4. List UI: `SearchField` Enter + history key, `ResultLane` + matching skeleton, `Stats` from that query’s `*Stats` extras (`href`/`note` allowed). Nested org: مؤسسة. باقة / اشتراك product names.
5. Detail UI: `FlexContainer dir="column" flx_1`. No heading on overlay. `PageStateLane`: loading (`Loadable` 3rem), `Empty`, `LaneFailed`. Forms: `FormStateLane` alias; heading/actions **outside** the lane. Do not edit `Wrong` / `Loadable` APIs. Overlay flags on the hook. Authority: `docs/platforms/cpanel/supervisor-state-lanes.md`.
6. Keep `SupervisorMainLayout` content as a `Col` so `flx_1` fills under chrome.
7. Refresh GQL mirrors by command copy. `ar.ts` only. No Forms keys unless the task ships a requester (plans: `SUPERVISOR_PLAN`; settings: `SUPERVISOR_SETTINGS`).
8. Home is not this skill — use `cpanel-supervisor-home`. Update C17 docs. Verify `cpanel` `yarn type-check`.
