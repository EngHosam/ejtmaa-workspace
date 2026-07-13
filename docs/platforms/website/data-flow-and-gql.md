# Website Data Flow and GQL (Ejtmaa)

Customer portal contract (see `overview.md`).

## Read path

1. Route page mounts adapter hook or `$$_dataAdapter` query.
2. Adapter calls `/website/data_adapters/gql` with customer GQL operation.
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
| customer | `customer`, `notification` |

## GQL project config

`graphql.config.yml` — projects for `base` and `customer` mirrors only.

Sync from backend:

- SDL: `backend/src/app/gql/definitions/customer.graphql`, `base.graphql`, `shared.graphql`
- Types: `backend/src/app/gql/gql-types/customer.ts`, `base.ts`

## Socket

- Namespace: `customer` (authed; selected in `src/app/services/socket.ts` when `authedAs === "CUSTOMER"`)
- Event: `onCustomerEventDate` (payload `OnCustomerEventDate { type: "UPDATED" }`, see `src/types/events.ts` and `src/resources/configs/socket/events.ts`)

## SSR boot

1. Server calls `/website/custom/start` (`API.CUSTOM.START`).
2. `global.setServerStartData(...)` hydrates auth.
3. Client prepares socket and marks started.

The `CUSTOM.START`/`CUSTOM.SELECT` endpoints ship in `api.ts`; the boot wiring that calls them is planned. See `docs/platforms/website/ssr-boot-and-startup.md`.

## Adapter enterMode

List and home adapters use `enterMode` on `useShallowAdapter` / `$$_dataAdapter` to control mount-time reload behavior.

| Mode | Behavior |
|---|---|
| `FORCE_RELOAD_ON_MOUNT` | Default for list routes and customer home/dashboard adapters. Adapter reloads on every route enter. |
| `LOAD_ON_MOUNT` | Exception for routes that must preserve in-memory adapter state across quick back-navigation within the same session. |

Rules:

- Customer home/dashboard hooks (`useCustomerDashboard`) and paginated list hooks (`useCustomerNotifications`) use `FORCE_RELOAD_ON_MOUNT`.
- List adapters document `enterMode` on the owning route page when a session-preserve exception applies.

The hooks above (`useCustomerDashboard`, `useCustomerNotifications`) are planned; no list/home adapter hooks ship yet. `initDataAdaptersProps` currently registers `DATA_ADAPTERS.ADAPTER1` (customer) with no list adapter wired. Governance: `.cursor/rules/website-list-adapter-enter-mode.mdc`.

## Related

- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
