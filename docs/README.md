# Ejtmaa Documentation

## Purpose

This documentation is the authoritative implementation reference for the **Ejtmaa** (اجتماع) project.
It is written to make adding, updating, and fixing platform behavior predictable and safe across the current workspace surfaces.

Primary goals:
- Make the dominant project patterns explicit.
- Keep architectural drift low while the project is still early-stage.
- Prioritize the `requesters` + `gql` workflow as the default execution model.
- Document constraints, invariants, and extension playbooks.

## Governance Note

This docs set is aligned with the always-on digest in `.cursor/rules/constitution.mdc`
and the full reference in `docs/governance/constitution-full.md`.
Per constitutional policy, persisted docs under `/docs` are English-only.

## Reference Baseline

Patterns are grounded in:
- `backend/src` for backend behavior and contracts.
- `docs/platforms/cpanel/` for the supervisor SSR contract.
- `docs/platforms/website/` for the customer SSR contract.
- Brand/theme source of truth: `website/src/resources/configs/theme.ts` (navy `#0B2057`, orange `#EC6901`).

## Documentation Map

- `docs/workspace-overview.md` — Workspace structure and platform boundaries.
- `docs/governance/constitution-full.md` — Full workspace constitution reference.
- `docs/design-color-system.md` — Complete brand/theme reference extracted from `theme.ts`.
- `docs/design-system/colors.md` — Color usage summary and semantic paths.

### Backend

- `docs/platforms/backend/overview.md` — Architecture, runtime, active domain surface.
- `docs/platforms/backend/README.md` — Contract and pattern index.
- `docs/platforms/backend/contracts/http-and-requesters.md` — HTTP mounts and requester matrix.
- `docs/platforms/backend/contracts/graphql-and-types.md` — Customer + supervisor GraphQL.
- `docs/platforms/backend/contracts/client-portal-http-website.md` — `/website` client portal.
- `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md` — `/external` payment callback target contract.
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md` — Supervisor admin read surfaces.
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md` — Supervisor customer reads and stats.
- `docs/platforms/backend/contracts/socket-event-mirroring.md` — Socket event mirror contract.
- `docs/invariants/backend.md` — Backend invariants.

### CPanel (Supervisor)

- `docs/platforms/cpanel/README.md` — Cpanel documentation index.
- `docs/platforms/cpanel/overview.md` — Supervisor SSR foundation.
- `docs/platforms/cpanel/ui-foundation.md` — Utils + theme.ts UI contract.
- `docs/platforms/cpanel/supervisor-admin-modules.md` — Route catalog.
- `docs/platforms/cpanel/data-flow-and-gql.md` — Adapters, requesters, GQL.
- `docs/platforms/cpanel/customer-management.md` — Customer list and detail.
- `docs/invariants/cpanel.md` — Cpanel invariants.

### Website (Customer)

- `docs/platforms/website/README.md` — Website documentation index and flow table.
- `docs/platforms/website/overview.md` — Customer portal foundation.
- `docs/platforms/website/route-registry-contract.md` — Customer route registry.
- `docs/platforms/website/data-flow-and-gql.md` — Adapters, requesters, GQL.
- `docs/platforms/website/ui-foundation.md` — Utils + theme.ts UI contract.
- `docs/invariants/website.md` — Website invariants.

## Platform Scope

| Platform | Role | HTTP mount |
|---|---|---|
| `backend/` | API, requesters, ORM, GQL | `/website`, `/cpanel` |
| `cpanel/` | Supervisor SSR | `/cpanel` |
| `website/` | Customer SSR | `/website` |

Actors:

- Backend middleware: **visitor**, **customer**, **supervisor**
- Website UI: **visitor**, **customer**
- Cpanel typed actor: **SUPERVISOR** (visitor scope for login forms)
