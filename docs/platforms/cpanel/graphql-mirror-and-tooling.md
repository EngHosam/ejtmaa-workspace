# CPanel GraphQL Mirror and Tooling

## Purpose

GraphQL mirror and tooling contract for `cpanel/` supervisor reads.
It exists so future cpanel and backend work can extend the same contract without drifting on:

- backend source-of-truth ownership,
- mirrored SDL and generated type placement,
- local GraphQL tooling config placement,
- and verification expectations.

## 1) Source of truth

The backend remains the only source of truth for the shared cpanel GraphQL contract.

Authoritative backend paths:

- `backend/src/app/gql/definitions/base.graphql`
- `backend/src/app/gql/definitions/supervisor.graphql`
- `backend/src/app/gql/gql-types/base.ts`
- `backend/src/app/gql/gql-types/supervisor.ts`

`cpanel/` mirrors copy from the backend paths listed below when the repository checkout includes GQL tooling.

## 2) Cpanel mirror surfaces

### 2.1 SDL mirrors

The local cpanel schema mirror contract targets:

- `cpanel/src/types/gql/definitions/base.graphql`
- `cpanel/src/types/gql/definitions/supervisor.graphql`

These files are mirror artifacts, not cpanel-owned schema authoring surfaces.

### 2.2 Generated type mirrors

The local generated GraphQL type mirror contract targets:

- `cpanel/src/types/gql/gql-types/base.ts`
- `cpanel/src/types/gql/gql-types/supervisor.ts`

These files are copied/generated contract artifacts and must not be hand-edited.

## 3) Local GraphQL tooling scope

The root cpanel schema project remains declared in:

- `cpanel/graphql.config.yml`

Local subtree helper configs exist in the two approved GraphQL-owning UI subtrees:

- `cpanel/src/app/ui/pages/graphql.config.yml`
- `cpanel/src/app/ui/components/graphql.config.yml`

Role of those helper files:

- scope editor/tooling schema awareness to the local subtree,
- point UI-facing GraphQL work at the mirrored `base` + `supervisor` schema pair,
- and avoid scattering helper configs across unrelated UI folders.

They do **not** declare independent schemas and they do **not** replace the central mirror under `src/types/gql/`.

## 4) Copy workflow and editing discipline

The mirror workflow is:

1. backend files remain the source of truth,
2. cpanel mirrors are synced by command-based copy,
3. copied SDL and copied/generated `gql-types` are not hand-edited inside `cpanel/`,
4. local `graphql.config.yml` files are added only in UI subtrees that actually host GraphQL-aware code.

This keeps cpanel aligned with the existing workspace GraphQL mirror discipline already used by `website/`.

## 5) Supervisor GQL operations in cpanel

Customer module adapters use mirrored supervisor operations:

- `customers.graphql` -- paginated customer list
- `customer.graphql` -- single customer read
- `customerStats.graphql` — `customerStats.total_count` on `HomeStatCard` and list header

Mirror SDL and gql-types under `cpanel/src/types/gql/` stay command-synced from backend; UI subtrees host local `graphql.config.yml` helpers only.

## 6) Verification

Verification uses the existing cpanel script:

- `yarn run type-check`

No new verification tooling was introduced.

## 7) Contract inventory

| Path | Contract role | Described in |
|------|---------------|--------------|
| `cpanel/graphql.config.yml` | Root schema project config | §3 |
| `cpanel/src/types/gql/definitions/base.graphql` | Copied SDL mirror from backend | §2.1 |
| `cpanel/src/types/gql/definitions/supervisor.graphql` | Copied SDL mirror from backend | §2.1 |
| `cpanel/src/types/gql/gql-types/base.ts` | Copied/generated type mirror from backend | §2.2 |
| `cpanel/src/types/gql/gql-types/supervisor.ts` | Copied/generated type mirror from backend | §2.2 |
| `cpanel/src/app/ui/pages/graphql.config.yml` | Local GraphQL tooling helper scoped to `pages/` | §3 |
| `cpanel/src/app/ui/components/graphql.config.yml` | Local GraphQL tooling helper scoped to `components/` | §3 |
| `cpanel/lib/tsconfig.tsbuildinfo` | Type-check verification artifact | §6 |

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/cpanel/repository-inventory.md`
- `docs/platforms/cpanel/component-structure.md`
- `docs/platforms/cpanel/data-flow-and-gql.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
