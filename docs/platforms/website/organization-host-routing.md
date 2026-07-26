# Website Organization Host Routing

## Scope

Host-based split of the SSR customer portal (`website/`) into two boots:

- **Apex (main) host** — visitor landing plus the authed `/customer/*` workspace.
- **Organization host** — a tenant surface reached through `{subdomain}.ejtmaa.live` or an organization custom domain.

The page covers host classification, the organization boot request, the `organizationHost` store slice, transport identification (`axios` header; meeting handshake query on `/meeting`), the `orgHostOnly` route gate, and the backend surfaces that answer them (`POST /website/custom/org/start`, socket namespace `/meeting` for Meeting realtime).

Backend HTTP contract: `docs/platforms/backend/contracts/client-portal-http-website.md`.
Backend socket contract: `docs/platforms/backend/contracts/meeting-realtime-socket.md`.
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

- `socket.prepareSocket` returns without creating a socket (see §4),
- Meeting realtime is owned later by `useMeetingSocket` on the Meeting page (§5.1),
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

Route gating and Meeting socket config use the **store** predicate; only `organization-host.start` uses the request classifier. Shared boot `prepareSocket` also uses the store predicate to **skip** creating a socket on an organization host.

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
| Shared boot socket | `website/src/resources/configs/socket.ts` → `opts.query` | `token` only (authed customer `/customer`); **no** `organizationId` |
| Meeting socket | `website/src/resources/configs/meeting-socket.ts` → `opts.query` | required `memberId`, `memberToken`, `meetingId`; optional `organizationId` from `state.organizationHost.id` when present |

Query objects are built before the config literal; absent values are omitted rather than sent as the string `"undefined"`.

Config layout: root entry `configs/socket.ts` is the shared boot factory; `configs/socket/events.ts` is a boot fragment; Meeting uses a **sibling root** entry `configs/meeting-socket.ts` (not nested under `socket/`).

## 4) Shared boot socket (`prepareSocket`)

`website/src/app/services/socket.ts` `prepareSocket`:

1. organization host → **return** (no boot socket),
2. else not authed → return,
3. else authed customer → namespace `customer` (`throw "wrong namespace"` when the actor is not `CUSTOMER`),
4. `createSocketInstance(getSocketInstanceConfig(..., "customer"))` + `asyncConnect`; expose via `myInstance.getSocket`.

`URLS(...).SOCKET_URL(namespace)` produces the connection URL as before. Meeting does **not** use `getSocket`.

## 5) Meeting route (only organization-host route today)

| Concern | Shipped |
|---|---|
| Identify / path | `Meeting` → `/meeting/:memberId/:memberToken/:meetingId` |
| `MPagesRoutes` | `params: { memberId: string; memberToken: string; meetingId: string }` |
| Flags | `layout: "MEETING"`, `orgHostOnly: true`, `preLoadedPage` |
| Layout | `website/src/app/ui/layouts/MeetingLayout.tsx` — `Box` with `semanticColor.pageBackground`, `minH="100vh"`; mapped in `MyApp.getLayout()` under `case "MEETING"` |
| Page | `website/src/app/ui/pages/Meeting.tsx` — `useCurrentParams({ ident: "Meeting", mapParams: p => p })` + `useMeetingSocket({ memberId, memberToken, meetingId })`; temporary UI shows `connected` / `disconnected` |
| Socket hook | `website/src/app/ui/components/meeting/hooks/useMeetingSocket.ts` (product hook under `components/meeting/`, **not** `ui/base/hooks`) |

`Layout` in `website/src/types/extends/global.ts` includes the `"MEETING"` member; `PageRouteState` includes `orgHostOnly?: boolean`.

### 5.1) Meeting socket session (`useMeetingSocket` + `/meeting`)

Meeting realtime is a **hook-owned** session on namespace `/meeting`, independent of the shared boot socket (§4).

`website/src/app/ui/components/meeting/hooks/useMeetingSocket.ts`:

1. Gate on CSR readiness (`useClientPreparing`) and require `memberId`, `memberToken`, `meetingId`.
2. `createSocketInstance(getMeetingSocketInstanceConfig(...))` with handshake query from `meeting-socket.ts` (§3).
3. Expose `{ connected, socket }` immediately after create (socket may still be disconnected; `connected` is the readiness flag).
4. Native `socket.on("connect")` → set `connected` and emit **`meeting.join`** with `{ meetingId, memberId, memberToken }`.
5. Native `socket.on("disconnect")` → clear `connected`; if reason is `io server disconnect`, call `socket.connect()` (Socket.IO does not auto-reconnect after a server-forced drop).
6. Open the session with `socket.connect()`.
7. Cleanup on unmount / deps change: `off` → `disconnect` → clear state.

Backend pairing (`docs/platforms/backend/contracts/meeting-realtime-socket.md`): handshake auth is `MeetingAuthenticationIOMiddleware` (`meeting_auth`); connection joins `Rooms.MEETING(meetingId)` and returns `["meeting.join"]`; `MeetingJoinIOController` is log-only and returns `[]` (listener unbound until the next connection bind). Re-emit on `connect` is the supported re-join after reconnect.

**Not mirrored:** `meeting.join` is inbound client → server. Do not add it to `website/src/types/events.ts` / socket event registries.

Authority: `.cursor/rules/meeting-realtime-socket.mdc`, `.cursor/skills/meeting-realtime-socket/SKILL.md`.

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

### 6.3 Socket `/meeting`

Full backend contract: `docs/platforms/backend/contracts/meeting-realtime-socket.md`.

What the website side depends on:

- namespace `/meeting`, registered in `backend/src/resources/configs/socket/io.ts` with `globalMiddlewares: ["meeting_auth"]`,
- `MeetingAuthenticationIOMiddleware` proves member token + meeting + roster row + `ACTIVE` organization before any controller runs, so a refused handshake surfaces as a connect error, not a partial session,
- `MeetingConnectionIOController` joins `Rooms.MEETING(meetingId)` and binds `["meeting.join"]`,
- `MeetingJoinIOController` returns `[]`, so the client must re-emit `meeting.join` on every `connect` (§5.1).

## 7) Failure modes

| Scenario | Behavior |
|---|---|
| Empty/unparsable `Host` on SSR | `resolveRequestHost` throws `"unexpected"` → server errors funnel |
| Organization host with unknown subdomain/custom domain | `org/start` returns `404` → errors funnel redirects to `Error` with `404` |
| Production `org/start` body without `subdomain` and without `customDomain` | `NOT_PERMIT` → `Error` with `403` |
| Non-production without `TEST_ORGANIZATION_ID` | `org/start` `404` |
| Apex request to `/meeting/...` | route gate → `Error` `404` |
| Organization-host request to any non-`orgHostOnly` route (`/`, `/login`, `/customer/*`) | route gate → `Error` `404` |
| Socket `/meeting` handshake missing ids, bad token, org mismatch, or no roster row | `NOT_VALID_CREDENTIAL` |

## 8) Known limits (shipped state, intentional)

1. **`org_host` is registered but unwired.** No HTTP route consumes `currentOrganization` yet; it is attached when organization-scoped endpoints land.
2. **`Meeting` UI is a connection-status probe only** (`connected` / `disconnected` text). No meeting content, media, or participant UX ships yet.
3. **`MeetingJoinIOController` is log-only** and returns `[]`, so `meeting.join` is one-shot per connection until reconnect re-binds it via `MeetingConnectionIOController`.
4. **Non-production organization resolution ignores the request body** and always uses `TEST_ORGANIZATION_ID`, so local runs exercise a single organization.
5. **Handshake values travel primarily on the Socket.IO query** built in `meeting-socket.ts`. Header names are also read on the server (`headers.x || query.x`), but Node lowercases headers; do not rely on camelCase header-only delivery.
6. **`meeting.join` is not mirrored** into frontend event registries — it is inbound only. Outbound meeting notify events, when added, still follow `socket-event-mirroring.md`.
7. **Organization-host pages other than `Meeting` have no realtime channel.** A tenant-wide org socket is not part of the shipped surface.

## 9) Environment

| Variable | Repo | Purpose |
|---|---|---|
| `TEST_ORGANIZATION_ID` | `backend/.env.example` (`=1`) | Non-production organization used by `org/start` |
| `NODE_ENV` | website runtime (`myInstance.getEnv`) | Enables `localhost` / `127.0.0.1` as an organization host |
| `SHARED__TEST_MODE` | website runtime | Unchanged; still selects local backend/socket URLs |

## 10) Traceability

Every path that implements this contract, with the section that describes it.

### Website (`website/`)

| Path | Role | Documented in |
|---|---|---|
| `src/app/helpers/RequestHostHelper.ts` | new — `resolveRequestHost`, `RequestHostResult` | §2 |
| `src/resources/configs/server.ts` | `extendMyInstance` exposes `resolveRequestHost` | §1.1, §2 |
| `src/app/services/organization-host.ts` | new — `isOrganizationHost`, `start` | §1.1, §2 |
| `src/app/services/index.ts` | server boot branches on `organizationHost.start` | §1.1 |
| `src/app/services/router.ts` | `orgHostOnly` gate before auth branches | §1.2 |
| `src/app/services/socket.ts` | org host → no boot socket; customer → `/customer` | §4 |
| `src/resources/configs/socket.ts` | boot query: `token` only (no `organizationId`) | §3, §4 |
| `src/resources/configs/meeting-socket.ts` | Meeting handshake factory → `SOCKET_URL("meeting")` | §3, §5.1 |
| `src/resources/configs/axios.ts` | `organizationId` request header | §3 |
| `src/resources/configs/axios/api.ts` | `CUSTOM.ORG_START` | §1.1, §6.1 |
| `src/resources/configs/store/reduces/organization-host.ts` | new slice | §3 |
| `src/resources/configs/store/reduces/index.ts` | slice registration + `MDefaultStoreState` | §3 |
| `src/resources/configs/routes.ts` | `Meeting` route with `orgHostOnly`; nested `MPagesRoutes` params for Meeting + fixed customer param routes | §5; `route-registry-contract.md` §3.1 |
| `src/types/extends/global.ts` | `resolveRequestHost` on `MyInstance`; `orgHostOnly`; `Layout` `"MEETING"` | §2, §5 |
| `src/app/ui/pages/Meeting.tsx` | params + `useMeetingSocket`; temporary connected status UI | §5, §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingSocket.ts` | hook-owned `/meeting` session + `meeting.join` + reconnect | §5.1 |
| `src/app/ui/layouts/MeetingLayout.tsx` | `MEETING` layout | §5 |
| `src/app/ui/base/core/MyApp.tsx` | `case "MEETING"` → `MeetingLayout` | §5 |
| `src/resources/configs/customer/formRoute.ts` | nested `params` href builders without `as To` | `route-registry-contract.md` §3.1 |
| `src/app/ui/components/customer/members/CustomerMemberFormScreen.tsx` | `useCurrentParams` `mapParams: p => p` | `route-registry-contract.md` §3.1 |
| `src/app/ui/components/customer/message-channels/CustomerMessageChannelFormScreen.tsx` | same | §3.1 |
| `src/app/ui/components/customer/message-templates/CustomerMessageTemplateFormScreen.tsx` | same | §3.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | same | §3.1 |
| `src/app/ui/components/customer/hooks/useCustomerMeetingDetails.ts` | same | §3.1 |
| `src/app/ui/components/customer/hooks/useCustomerHome.ts` | typed `To<"CustomerMeetingDetails">` without cast | §3.1 |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | details href without cast | §3.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingCard.tsx` | typed details href | §3.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingFormScreen.tsx` | details `nav.push` without cast | §3.1 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Backend (`backend/`)

| Path | Role | Documented in |
|---|---|---|
| `src/app/http/controllers/website/custom/OrgStartController.ts` | org start payload + resolution | §6.1 |
| `src/app/http/routes/website.ts` | `OrgCustomRouter` + `POST /custom/org/start` | §6.1 |
| `src/app/http/middlewares/OrganizationHostMiddleware.ts` | header-based org resolve + `currentOrganization` | §6.2, §8 |
| `src/resources/configs/http/middlewares/index.ts` | `org_host` registration | §6.2 |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | actor handshake `token` + `SocketData` for `/customer`, `/supervisor` | `../backend/contracts/meeting-realtime-socket.md` §2 |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | `/meeting` handshake + `MeetingSocketData` + `current*` helpers | `../backend/contracts/meeting-realtime-socket.md` §2 |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | join `Rooms.MEETING`; return `["meeting.join"]` | `../backend/contracts/meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingJoinIOController.ts` | log-only `meeting.join`; return `[]` | `../backend/contracts/meeting-realtime-socket.md` §3, §6 |
| `src/resources/configs/socket/io.ts` | `/meeting` namespace + `meeting_auth` + meeting controllers | `../backend/contracts/meeting-realtime-socket.md` §1 |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM / room | `../backend/contracts/meeting-realtime-socket.md` §4 |
| `.env.example` | `TEST_ORGANIZATION_ID=1` | §9 |

### Governance

| Path | Role |
|---|---|
| `.cursor/rules/organization-host-routing.mdc` | host mode, route gate, transport identification |
| `.cursor/rules/meeting-realtime-socket.mdc` | Meeting session placement, hook contract, backend pairing |
| `.cursor/rules/nodejs-socket-namespace-registration.mdc` | namespace / controller / room registration |
| `.cursor/rules/nodejs-socket-handler-contract.mdc` | connection return = absolute listener set |
| `.cursor/rules/socket-event-mirroring.mdc` | outbound mirror scope |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | Meeting realtime workflow |
| `.cursor/skills/nodejs-socket-server-event/SKILL.md` | backend socket surface workflow |

## 11) Verification

- `yarn type-check` in `website/` and in `backend/`.
- Apex host: `/` boots through `API.CUSTOM.START`; `/meeting/...` renders `Error` `404`.
- Organization host: boot calls `org/start` and hydrates `organizationHost`; `/meeting/...` shows temporary connected/disconnected status on `MEETING` layout; `/customer/...` renders `Error` `404`.
- Socket: organization host opens **no** boot socket; Meeting page opens `/meeting` via `useMeetingSocket`, joins `meeting-{id}`, and emits `meeting.join` (server log); apex authed customer still connects to `/customer`.
- Forced server disconnect: client reconnects via `useMeetingSocket` and re-emits `meeting.join`.

## 12) Related

- `docs/platforms/website/ssr-boot-and-startup.md` — boot phases
- `docs/platforms/website/route-registry-contract.md` §3.1, §5.4 — nested params + organization-host route block
- `docs/platforms/backend/contracts/client-portal-http-website.md` — `/website` mount contract
- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — `/meeting` namespace, handshake auth, rooms
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket namespaces, rooms, `meeting.join`
- `docs/platforms/backend/modules/nodejs-socket-library.md` §10 — `/meeting` child events
- `docs/invariants/website.md` W58
- `docs/invariants/backend.md` B24
- `.cursor/rules/organization-host-routing.mdc`
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/website-mpages-routes-params-contract.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
