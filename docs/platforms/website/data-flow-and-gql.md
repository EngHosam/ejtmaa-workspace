# Website Data Flow and GQL (Ejtmaa)

Customer portal contract (see `overview.md`).

## Read path

1. Route page mounts adapter hook or `$$_dataAdapter` query.
2. Authed customer adapters call `/website/data_adapters/customer/gql` (`API.DATA_ADAPTERS.CUSTOMER.GQL`); visitor/global reads use `/website/data_adapters/gql`.
3. Backend `GQLAdapterController` resolves through `CustomerSchema` bridges.
4. Response typed from mirrored `src/types/gql/gql-types/customer.ts`.

## Write path

1. Route page binds form to requester via `API.FORMS.R` (visitor) or `API.FORMS.CUSTOMER.R` (customer).
2. Visitor form controller or `CustomerRequesterController` dispatches to the requester.
3. Requester validates, authorizes, persists, emits side effects (notify/socket).

Shipped endpoint map (`src/resources/configs/axios/api.ts`):

| Key | Path |
|---|---|
| `DATA_ADAPTERS.GQL` | `/data_adapters/gql` |
| `DATA_ADAPTERS.CUSTOMER.GQL` | `/data_adapters/customer/gql` |
| `FORMS.R` (visitor) | `/forms/requester/{requester}/{sub}` |
| `FORMS.CUSTOMER.R` | `/forms/customer/requester/{requester}/{sub}` |
| `ACTIONS.LOGOUT` | `/actions/logout` |
| `ACTIONS.MULTIPART_UPLOAD` | `/actions/multipart_upload` |
| `CUSTOM.START` | `/custom/start` |
| `CUSTOM.SELECT` | `/custom/select` |

## Actor maps

Website requester typing (`src/types/requesters/requesters.website.ts`):

| Actor | Requesters |
|---|---|
| visitor | `auth` |
| customer | `customer`, `member`, `organization`, `notification`, `subscription` |

## GQL project config

`graphql.config.yml` — projects for `base` and `customer` mirrors only.

Sync from backend:

- SDL: `backend/src/app/gql/definitions/customer.graphql`, `base.graphql`, `shared.graphql`
- Types: `backend/src/app/gql/gql-types/customer.ts`, `base.ts`

## Socket

- Namespace: `customer` (authed; selected in `src/app/services/socket.ts` when `authedAs === "CUSTOMER"`)
- Event: `OnCustomerEvent` (payload `OnCustomerEventDate { type: "UPDATED" }`, see `src/types/events.ts` and `src/resources/configs/socket/events.ts`)

## SSR boot

1. Server calls `/website/custom/start` (`API.CUSTOM.START`).
2. `global.setServerStartData(...)` hydrates auth.
3. `router.setRouterAccessPermission(...)`.
4. When `authedAs === "CUSTOMER"`, `auth.loadCurrentCustomer` → `LoadCurrentCustomer` hydrates `CUSTOMER_ME` for SSR shell.
5. Client prepares socket and marks started.

See `docs/platforms/website/ssr-boot-and-startup.md` and `flow-customer-shell.md` §6.

## Customer `me` adapter

| Adapter | API | Core query |
|---|---|---|
| `DATA_ADAPTERS.CUSTOMER_ME` | `API.DATA_ADAPTERS.CUSTOMER.GQL` | `me { id name avatar_url organization { id name logo_url subdomain } currentSubscription { id status plan_price plan_billing_period plan_max_* starts_at ends_at plan { id name } } }` |

Do not point `CUSTOMER_ME` at visitor/global `DATA_ADAPTERS.GQL`. Hook: `website/src/app/ui/components/customer/hooks/useMe.tsx`. Organization for the portal is nested under `_Me` only (no customer root `Query.organization`). Core query feeds shell marks + router org setup gate (`flow-customer-organization.md` §3.1).

## Customer GQL inherit base + members list

| Adapter | API | Role |
|---|---|---|
| `DATA_ADAPTERS.CUSTOMER_GQL` | `API.DATA_ADAPTERS.CUSTOMER.GQL` | Inherit target for mount-private customer list adapters |
| `"customer-members"` (private id) | inherits `CUSTOMER_GQL` | Members directory — `useCustomerMembers` |

Members query uses `listable: "members"` and optional `filter: { search }` from route history key `members`. Full contract: `flow-customer-members.md`.

Member writes use `Forms.CUSTOMER_MEMBER` → `API.FORMS.CUSTOMER.R("member")(sub)` (`read` | `create` | `update` | `delete`). Avatar binary upload uses `API.ACTIONS.MULTIPART_UPLOAD` before storing the filename in form `avatar_file`. See `flow-form-foundation.md` and `member-domain.md` §9.

Organization settings writes use `Forms.CUSTOMER_ORGANIZATION` → `API.FORMS.CUSTOMER.R("organization")(sub)` (`read` | `upsert`). Logo upload reuses the same multipart helper into form `logo_file` via `FormAvatarField` `name`. See `flow-customer-organization.md` and `organization-domain.md` §9.

Default `initDataAdaptersProps.default.maxLoadLength` is **24** (shared load-more page size for adapters that do not override it).

## Adapter enterMode

List and home adapters use `enterMode` on `useShallowAdapter` / `$$_dataAdapter` to control mount-time reload behavior.

| Mode | Behavior |
|---|---|
| `FORCE_RELOAD_ON_MOUNT` | Default for list routes and customer home/dashboard adapters. Adapter reloads on every route enter. |
| `LOAD_ON_MOUNT` | Exception for routes that must preserve in-memory adapter state across quick back-navigation within the same session. |

Rules:

- `useMe` default path uses `LOAD_ON_MOUNT` (optional `updateOnEnter` → `FORCE_RELOAD_ON_MOUNT`).
- Planned list/home hooks (`useCustomerDashboard`, `useCustomerNotifications`) use `FORCE_RELOAD_ON_MOUNT` when added.
- List adapters document `enterMode` on the owning route page when a session-preserve exception applies.
- Shipped `useCustomerMembers` reloads via `useEffect` → `mLoad({ reload: true, query: adapterQuery })` when the history-derived query changes (covers remount + search commit without a separate `enterMode` flag).

Shipped customer adapters in `initDataAdaptersProps`: `ADAPTER1`, `CUSTOMER_ME`, `CUSTOMER_GQL`. Governance: `.cursor/rules/website-list-adapter-enter-mode.mdc`, `.cursor/rules/website-customer-list-history-search.mdc`.

## Related

- `docs/platforms/website/flow-customer-members.md`
- `docs/platforms/website/flow-customer-organization.md`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/member-domain.md`
