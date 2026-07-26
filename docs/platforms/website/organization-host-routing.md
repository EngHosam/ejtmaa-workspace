# Website Organization Host Routing

## Scope

Host-based split of the SSR customer portal (`website/`) into two boots:

- **Apex (main) host** — visitor landing plus the authed `/customer/*` workspace.
- **Organization host** — a tenant surface reached through `{subdomain}.ejtmaa.live` or an organization custom domain.

The page covers host classification, the organization boot request, the `organizationHost` store slice, transport identification (`axios` header and socket query), the `orgHostOnly` route gate, and the backend surfaces that answer them (`POST /website/custom/org/start`, socket namespace `/org`).

Backend contract: `docs/platforms/backend/contracts/client-portal-http-website.md`.
Socket runtime: `docs/platforms/backend/modules/runtime-integrations.md` §5.

## 1) End-to-end flow

### 1.1 SSR request

1. `website/src/resources/configs/server.ts` extends the instance in `ServerConfig.extendMyInstance`, exposing `myInstance.resolveRequestHost()`.
2. `website/src/app/services/index.ts` `boot.server`:
   - early-exits on the `Error` route,
   - calls `organizationHost.start(myInstance)`,
   - **returns immediately when that call resolves `true`** (organization host),
   - otherwise continues the apex boot: `API.CUSTOM.START` → `global.setServerStartData` → `router.setRouterAccessPermission` → `auth.loadCurrentCustomer`.
3. `website/src/app/services/organization-host.ts` `start`:
   - resolves the request host once,
   - returns `false` for `{ isMain: true }` (no request issued),
   - otherwise `POST`s `API.CUSTOM.ORG_START` with a **classified** body — `{ subdomain }` or `{ customDomain }` — using `throwMode`, `throwWithErrorsFunnel`, `autoShowMainMessages`,
   - hydrates the response through `global.setServerStartData(myInstance, res.data)` and returns `true`.
4. The start payload writes the `organizationHost` reducer slice, which is the runtime host-mode signal for both SSR and CSR.

Classification happens **once on the website**; the backend never re-derives host semantics from the proxied `Host` header.

### 1.2 Route gate

`website/src/app/services/router.ts` `applyRouterMiddleware`, run from the `MyPage` lifecycle:

1. `Error` route early-exit (unchanged).
2. Read `orgHostOnly` from the current `routeState`.
3. Resolve host mode with `organizationHost.isOrganizationHost(myInstance)` (store-backed, works on SSR and CSR).
4. Mismatch in either direction → `redirect({ identify: "Error", params: { error: 404 }, replace: true })` followed by a `RESOLVED` `throwWebCoreError`:
   - organization host + route without `orgHostOnly`,
   - apex host + route with `orgHostOnly`.
5. `orgHost && orgHostOnly` → `return`: organization-host pages skip the authed-customer rules (org setup gate, public-route bounce, `canDoAction`), because the organization boot never loads customer auth state.
6. Everything else falls through to the existing authenticated/unauthenticated branches.

The gate runs **before** the auth branches; a wrong-host request never reaches a login redirect.

### 1.3 Client boot

`boot.client` is shared. On an organization host:

- `socket.prepareSocket` selects namespace `org` (see §4),
- `global.setClientIsStarted` and pre-message display behave as on apex,
- no customer permissions or `CUSTOMER_ME` load happens, because the server phase skipped them.

## 2) Host classification

`website/src/app/helpers/RequestHostHelper.ts`

```ts
export type RequestHostResult =
    | { isMain: true }
    | { isOrg: true, subdomain: string }
    | { isOrg: true, customDomain: string };
```

`resolveRequestHost(myInstance)` reads `myInstance.getReq()` (`headers.host` then `hostname`), normalizes (first comma segment, trim, lowercase, strip port), and throws `"unexpected"` when the result is empty. Decision order:

| Order | Condition | Result |
|---|---|---|
| 1 | `myInstance.getEnv("NODE_ENV") === "development"` and host is `localhost` / `127.0.0.1` | `{ isOrg: true, customDomain }` — local organization simulation |
| 2 | host is `192.168.1.10` | `{ isMain: true }` — local apex |
| 3 | host is `ejtmaa.live` or `www.ejtmaa.live` | `{ isMain: true }` |
| 4 | host matches `{sub}.ejtmaa.live` with a single non-reserved label (`www` and empty are reserved) | `{ isOrg: true, subdomain }` |
| 5 | anything else | `{ isOrg: true, customDomain }` |

The apex suffix comes from `ORG_PUBLIC_DOMAIN` in `website/src/resources/configs/urls.ts` — the same constant used by the organization settings subdomain field.

Environment is read through `myInstance.getEnv`, not `process.env`.

### Naming boundary (do not merge these two)

| Symbol | Source | Availability | Meaning |
|---|---|---|---|
| `myInstance.resolveRequestHost()` | request `Host` header | SSR only | How the browser reached us |
| `organizationHost.isOrganizationHost(myInstance)` | `state.organizationHost.id` | SSR + CSR | Whether the current boot hydrated an organization |

Route gating and socket selection use the **store** predicate; only `organization-host.start` uses the request classifier.

## 3) Store slice and transport identification

`website/src/resources/configs/store/reduces/organization-host.ts` registers the `organizationHost` slice in `store/reduces/index.ts` (`MDefaultStoreState.organizationHost`):

```ts
export type OrganizationHostReducerProps = {
    id?: string,
    name?: string,
    description?: string | null,
    logo_url?: string | null,
    primary_color?: string | null,
    secondary_color?: string | null
};
```

It spreads `action.organizationHost` on `GLOBAL.SSR_DID_START`, `GLOBAL.CSR_WILL_START`, and `GLOBAL.CSR_DID_START`; otherwise it defers to `reducersMixin`. The apex boot never writes it, so `id` stays undefined and every host predicate reads `false`.

Outgoing identification, sent only when a value exists:

| Transport | Where | Payload |
|---|---|---|
| HTTP | `website/src/resources/configs/axios.ts` → `ReqHeaderMiddleware.otherHeaders` | header `organizationId` from `state.organizationHost.id` |
| Socket | `website/src/resources/configs/socket.ts` → `opts.query` | `token` and/or `organizationId`, each added only when non-empty |

The socket query object is built before the config literal; absent values are omitted rather than sent as the string `"undefined"`.

## 4) Socket namespace selection

`website/src/app/services/socket.ts` `prepareSocket`:

1. organization host → namespace `org`,
2. else authed customer → namespace `customer` (`throw "wrong namespace"` when the actor is unknown),
3. else → return without creating a socket.

`URLS(...).SOCKET_URL(namespace)` produces the connection URL as before.

## 5) Meeting route (only organization-host route today)

| Concern | Shipped |
|---|---|
| Identify / path | `Meeting` → `/meeting/:memberId/:memberToken/:meetingId` |
| Flags | `layout: "MEETING"`, `orgHostOnly: true`, `preLoadedPage` |
| Layout | `website/src/app/ui/layouts/MeetingLayout.tsx` — `Box` with `semanticColor.pageBackground`, `minH="100vh"`; mapped in `MyApp.getLayout()` under `case "MEETING"` |
| Page | `website/src/app/ui/pages/Meeting.tsx` — reads `memberId` / `memberToken` / `meetingId` via `useCurrentParams` and renders `null` |

`Meeting` is a **route and layout shell**: the gate, boot, and namespace selection are exercised, but no meeting UI or data load ships in this change set. Any meeting surface is added inside this page later.

`Layout` in `website/src/types/extends/global.ts` gained the `"MEETING"` member; `PageRouteState` gained `orgHostOnly?: boolean`.

## 6) Backend surfaces

### 6.1 `POST /website/custom/org/start`

`backend/src/app/http/routes/website.ts` registers `OrgCustomRouter = CustomRouter("/org")()` — **no `auth` group middleware and no `org_host` middleware**, because the organization is resolved from the request body.

`backend/src/app/http/controllers/website/custom/OrgStartController.ts`:

- **Non-production** (`NODE_ENV !== "production"`): resolves the organization from `TEST_ORGANIZATION_ID` only and ignores the body; missing variable yields no organization → `404`.
- **Production**: resolves by `subdomain` or `customDomain` from the body, case-insensitively (`lower(cast(col(...), "text"))`) and requires `status === "ACTIVE"`; a body with neither key throws `NOT_PERMIT`.
- Success payload (public branding only):

```json
{
  "organizationHost": {
    "id": "…",
    "name": "…",
    "description": null,
    "logo_url": null,
    "primary_color": null,
    "secondary_color": null
  }
}
```

No auth, customer, or permission data is returned; the website consumes it as start data for the `organizationHost` slice.

### 6.2 `org_host` HTTP middleware

`backend/src/app/http/middlewares/OrganizationHostMiddleware.ts`, registered as `"org_host"` in `backend/src/resources/configs/http/middlewares/index.ts`:

- reads the `organizationid` request header (Express lowercases header names, so the website's `organizationId` header arrives here),
- loads the `ACTIVE` organization by id, throwing `404` when the header is missing or the organization does not resolve,
- attaches `req.organizationHostMiddleware = { organization }`,
- exports `currentOrganization(req, sure?)` for controllers.

**Not attached to any route in this change set** — see §8.

### 6.3 Socket `/org`

`backend/src/resources/configs/socket/io.ts` adds:

- controller key `org_connection` → `backend/src/app/socket/controllers/OrgConnectionIOController.ts`,
- namespace `/org` with global middleware `auth` and `connection: "org_connection"`.

`backend/src/app/socket/middlewares/AuthenticationIOMiddleware.ts` now has two linear paths:

1. **token present** — unchanged customer/supervisor resolution, then `this.next()` and `return`,
2. **no token** — reads `organizationId` from the handshake, loads the `ACTIVE` organization, throws `NOT_VALID_CREDENTIAL` when it does not resolve, and stores it as `socket.data.organization`. `isAuthed` is not set on this path.

`SocketData` gained `organization: OrganizationModel | undefined`.

`OrgConnectionIOController` refuses a connection without `socket.data.organization` and otherwise joins `Rooms.ORGANIZATION(id)`.

`backend/src/resources/consts/NotificationsConsts.ts` adds the matching entries:

| Map | Key | Value |
|---|---|---|
| `Namespaces` | `ORGANIZATION` | `/org` |
| `FCM_Namespaces` | `ORGANIZATION` | `-org` |
| `Rooms` | `ORGANIZATION(organizationId)` | `notification-organization-{id}` |

No org-scoped event is emitted yet; the namespace only accepts connections and room membership.

## 7) Failure modes

| Scenario | Behavior |
|---|---|
| Empty/unparsable `Host` on SSR | `resolveRequestHost` throws `"unexpected"` → server errors funnel |
| Organization host with unknown subdomain/custom domain | `org/start` returns `404` → errors funnel redirects to `Error` with `404` |
| Production `org/start` body without `subdomain` and without `customDomain` | `NOT_PERMIT` → `Error` with `403` |
| Non-production without `TEST_ORGANIZATION_ID` | `org/start` `404` |
| Apex request to `/meeting/...` | route gate → `Error` `404` |
| Organization-host request to any non-`orgHostOnly` route (`/`, `/login`, `/customer/*`) | route gate → `Error` `404` |
| Socket `/org` handshake without a resolvable organization | `NOT_VALID_CREDENTIAL` |

## 8) Known limits (shipped state, intentional)

1. **`org_host` is registered but unwired.** No HTTP route consumes `currentOrganization` yet; it is attached when organization-scoped endpoints land.
2. **`/org` socket authorization is organization-id only.** The id is public (returned by `org/start`), so the namespace currently proves *which* organization a client claims, not that the client is an authorized participant. Participant-level proof belongs to the meeting surface when it ships.
3. **`Meeting` renders nothing.** Route, layout, gate, and namespace are wired; the meeting experience is not.
4. **Non-production organization resolution ignores the request body** and always uses `TEST_ORGANIZATION_ID`, so local runs exercise a single organization.
5. **`handshake.headers.organizationId` is effectively inert.** Node lowercases header names, so the socket value is carried by the handshake **query** built in `website/src/resources/configs/socket.ts`.
6. **No organization socket events are mirrored**, so `docs/platforms/backend/contracts/socket-event-mirroring.md` is unchanged by this work.

## 9) Environment

| Variable | Repo | Purpose |
|---|---|---|
| `TEST_ORGANIZATION_ID` | `backend/.env.example` (`=1`) | Non-production organization used by `org/start` |
| `NODE_ENV` | website runtime (`myInstance.getEnv`) | Enables `localhost` / `127.0.0.1` as an organization host |
| `SHARED__TEST_MODE` | website runtime | Unchanged; still selects local backend/socket URLs |

## 10) Traceability

Every path in the change set, with the section that describes it.

### Website (`website/`)

| Path | Change | Documented in |
|---|---|---|
| `src/app/helpers/RequestHostHelper.ts` | new — `resolveRequestHost`, `RequestHostResult` | §2 |
| `src/resources/configs/server.ts` | `extendMyInstance` exposes `resolveRequestHost` | §1.1, §2 |
| `src/app/services/organization-host.ts` | new — `isOrganizationHost`, `start` | §1.1, §2 |
| `src/app/services/index.ts` | server boot branches on `organizationHost.start` | §1.1 |
| `src/app/services/router.ts` | `orgHostOnly` gate before auth branches | §1.2 |
| `src/app/services/socket.ts` | namespace selection `org` / `customer` / none | §4 |
| `src/resources/configs/socket.ts` | conditional `token` / `organizationId` query | §3, §4 |
| `src/resources/configs/axios.ts` | `organizationId` request header | §3 |
| `src/resources/configs/axios/api.ts` | `CUSTOM.ORG_START` | §1.1, §6.1 |
| `src/resources/configs/store/reduces/organization-host.ts` | new slice | §3 |
| `src/resources/configs/store/reduces/index.ts` | slice registration + `MDefaultStoreState` | §3 |
| `src/resources/configs/routes.ts` | `Meeting` route with `orgHostOnly` | §5 |
| `src/types/extends/global.ts` | `resolveRequestHost` on `MyInstance`; `orgHostOnly`; `Layout` `"MEETING"` | §2, §5 |
| `src/app/ui/pages/Meeting.tsx` | new — params-only shell | §5 |
| `src/app/ui/layouts/MeetingLayout.tsx` | new — `MEETING` layout | §5 |
| `src/app/ui/base/core/MyApp.tsx` | `case "MEETING"` → `MeetingLayout` | §5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Backend (`backend/`)

| Path | Change | Documented in |
|---|---|---|
| `src/app/http/controllers/website/custom/OrgStartController.ts` | new — org start payload + resolution | §6.1 |
| `src/app/http/routes/website.ts` | `OrgCustomRouter` + `POST /custom/org/start` | §6.1 |
| `src/app/http/middlewares/OrganizationHostMiddleware.ts` | new — header-based org resolve + `currentOrganization` | §6.2, §8 |
| `src/resources/configs/http/middlewares/index.ts` | `org_host` registration | §6.2 |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | org handshake path + `SocketData.organization` | §6.3 |
| `src/app/socket/controllers/OrgConnectionIOController.ts` | new — `/org` connection → org room | §6.3 |
| `src/resources/configs/socket/io.ts` | `/org` namespace + `org_connection` | §6.3 |
| `src/resources/consts/NotificationsConsts.ts` | `ORGANIZATION` namespace / FCM / room | §6.3 |
| `.env.example` | `TEST_ORGANIZATION_ID=1` | §9 |

## 11) Verification

- `yarn type-check` in `website/` and in `backend/`.
- Apex host: `/` boots through `API.CUSTOM.START`; `/meeting/...` renders `Error` `404`.
- Organization host: boot calls `org/start` and hydrates `organizationHost`; `/meeting/...` renders the `MEETING` layout; `/customer/...` renders `Error` `404`.
- Socket: organization host connects to `/org` and joins `notification-organization-{id}`; apex authed customer still connects to `/customer`.

## 12) Related

- `docs/platforms/website/ssr-boot-and-startup.md` — boot phases
- `docs/platforms/website/route-registry-contract.md` §5.4 — organization-host route block
- `docs/platforms/backend/contracts/client-portal-http-website.md` — `/website` mount contract
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket namespaces and rooms
- `docs/invariants/website.md` W58
- `.cursor/rules/website-org-host-routing.mdc`
