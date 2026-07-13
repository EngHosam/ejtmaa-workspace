# Supervisor Customers and Stats Contract

## 1) Scope

Supervisor-facing customer read surfaces that power cpanel customer administration:

- paginated customer list and single-customer read,
- aggregate customer count KPI,
- bridge and resolver ownership for those reads.

## 2) Supervisor GraphQL customer read surface

SDL: `backend/src/app/gql/definitions/supervisor.graphql`

Types and queries:

- `_Customer`
- `_CustomerFilter`
- `_CustomerSortValue`
- `customers(filter: _CustomerFilter): [_Customer]`
- `customer(id: ID!): _Customer`

`_Customer` fields:

- `id`
- `name`
- `email`
- `mobile`
- `avatar_file`
- `avatar_url`
- `total_count` (pagination)

`_CustomerFilter` inputs:

- `search`
- `email`
- `mobile`
- `sort`

`_CustomerSortValue` values:

- `UPDATED_AT_DESC`
- `UPDATED_AT_ASC`
- `NAME_ASC`
- `NAME_DESC`
- `EMAIL_ASC`
- `EMAIL_DESC`

## 3) Customer stats aggregate

Root query:

- `customerStats: _CustomerStats`

`_CustomerStats` fields:

- `total_count: Int!`

Loaded through `CustomerStatsBridge.loadExtra(...)` from `Customer().count()`.

Dashboard KPI cards use `customerStats.total_count`.

## 4) Bridge and resolver ownership

`CustomerBridge` owns supervisor customer list and detail reads:

- root `many`: listable pagination, search across `name` / `email` / `mobile`, filter narrowing, deterministic sort, secondary order by `id`,
- root `one`: lookup by `id`.

`CustomerStatsBridge` owns `customerStats.total_count`.

`SupervisorSchema` registers both bridges and forwards:

- `customers` → `prepareManyGQLModels({ filter })`
- `customer` → `prepareOneGQLModel({ id })`
- `customerStats` → `prepareOneGQLModel({})`

`MeBridge` serves supervisor profile only (`id`, `name`, `email` on `_Me`).

## 5) Traceability map

| Path | Role |
|------|------|
| `backend/src/app/gql/definitions/supervisor.graphql` | SDL source of truth |
| `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts` | List/detail ORM query construction |
| `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts` | `total_count` aggregate |
| `backend/src/app/gql/bridges/supervisor/MeBridge.ts` | Supervisor profile |
| `backend/src/app/gql/schemas/SupervisorSchema.ts` | Thin resolver wiring |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated types |

## Related

- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `docs/platforms/cpanel/customer-management.md`
