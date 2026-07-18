# Website Repository Inventory (Ejtmaa)

## Repository structure

`website/` is a separate git repository for the Customer SSR portal.

```
website/
├── src/
│   ├── app/
│   │   ├── services/          # auth, boot, socket, store
│   │   ├── ui/
│   │   │   ├── base/          # framework-bound infrastructure (Utils, hooks, core)
│   │   │   ├── components/    # reusable product UI
│   │   │   ├── layouts/       # shell layouts (BasicLayout, MainLayout)
│   │   │   └── pages/         # route page orchestration
│   │   └── helpers/
│   ├── client/                # client entry
│   ├── server/                # SSR entry
│   └── resources/
│       ├── configs/           # web-core, theme, utils, axios, routes, store
│       └── translations/      # ar.ts, en.ts
├── src/types/
│   ├── gql/                   # mirrored base + customer SDL/types
│   ├── extends/               # route metadata extensions
│   └── requesters/            # typed requester maps
└── graphql.config.yml
```

## Contract inventory (customer portal)

Paths below describe the customer portal contract inventory for `website/`.

## Key config files

| Path | Role |
|---|---|
| `src/resources/configs/theme.ts` | Brand source of truth (navy `#0B2057`, orange `#EC6901`) |
| `src/resources/configs/utils.ts` | Theme path resolution |
| `src/resources/configs/routes.ts` | Customer route registry |
| `src/resources/configs/web-core.ts` | `@my-ssr/web-core` bootstrap |
| `src/resources/configs/axios/api.ts` | `/website` endpoint map |
| `src/app/services/auth.ts` | Auth state and `AuthedAs` type |

## GQL mirrors

- `src/types/gql/definitions/base.graphql` — shared SDL (scalars, `_Ability`, `_Notification*`, interfaces, `_Notification`)
- `src/types/gql/definitions/customer.graphql` — customer SDL (`_Me`, `_Organization`, `_Member` + root `Query { me, notifications, organization, members, member }`)
- `src/types/gql/definitions/shared.graphql` — shared `Query { notifications }` extension stub
- `src/types/gql/gql-types/base.ts` — shared generated types
- `src/types/gql/gql-types/customer.ts` — customer generated types

Supervisor SDL/types are a `cpanel/` concern and are not mirrored under `website/`.

## Requesters

`src/types/requesters/requesters.website.ts` defines `RequestersMap`:

| Actor | Requester | Subs |
|---|---|---|
| `visitor` | `auth` | `registerCustomer`, `login`, `resetPassword` |
| `customer` | `customer` | `readSettings`, `updateSettings` |
| `customer` | `notification` | `deleteAll` |

## Boundaries

- Do not place business logic in `ui/base/` — infrastructure only.
- Product UI in `ui/components/` at top level or feature folders.
- Route pages orchestrate; extract only truly reusable sections.


## Related

- `docs/platforms/website/component-structure.md`
- `docs/platforms/website/overview.md`
