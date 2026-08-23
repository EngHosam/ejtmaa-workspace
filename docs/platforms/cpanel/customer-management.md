# CPanel Customer Management

## Status

**Not implemented in the CPanel frontend.**

This page records the **backend supervisor contract** that a future CPanel customers module must follow. The current `cpanel/` checkout has no customer list/detail routes, no customer table, and no home KPI cards.

Do not treat this document as a route catalog. Frontend paths and identifies are not chosen yet.

## Backend contract (already shipped on `/cpanel`)

Supervisor GQL:

- `customers` — paginated list
- `customer(id)` — single record
- `customerStats.total_count` — aggregate count

Requesters: supervisor `customer` read|update on the cpanel platform.

When a customers UI is added later, it must:

- register routes in `cpanel/src/resources/configs/routes.ts` through `supervisorRouter` (see `route-registry-contract.md`)
- use `DATA_ADAPTERS.SUPERVISOR_GQL` for reads and `API.FORMS.SUPERVISOR.R` for writes
- keep list search history-backed
- bind any KPI tiles to `customerStats.total_count` only (see `docs/invariants/cpanel.md` C18)

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/cpanel/supervisor-admin-modules.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
