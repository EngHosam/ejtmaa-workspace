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
- Meeting realtime is owned later by `MeetingLiveProvider` in `MeetingLayout` (§5.1),
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
| Layout | `website/src/app/ui/layouts/MeetingLayout.tsx` — `MeetingLiveProvider` → `MeetingLiveSessionProvider` → `MeetingPageProvider` → `MeetingShell` (linking gate §5.3, branded shell §5.4, in-shell page §5.5); mapped in `MyApp.getLayout()` under `case "MEETING"` |
| Page | `website/src/app/ui/pages/Meeting.tsx` — switches in-shell pages from `useMeetingPage().page` (§5.5); default `"init"` → `meeting/pages/MeetingInitPage` |
| In-shell page state | `useMeetingPage.tsx` — `MeetingPage` type + `MeetingPageProvider` / `useMeetingPage` (§5.5) |
| Live transport | `website/src/app/ui/components/meeting/hooks/useMeetingLive.tsx` — socket + Yjs + SyncedStore instance + provider + public context reader (product module under `components/meeting/`, **not** `ui/base/hooks`) |
| Live session surface | `hooks/useMeetingLiveSession.tsx` — resolve + provider → `{ linking, can, actions, meeting, me, attendWindow }` (§5.3) |
| Current participant | `website/src/app/ui/components/meeting/hooks/useMeetingLiveMe.ts` — SyncedStore proxy for the route member's roster entry (§5.2); product UI prefers `me` from session (§5.3) |
| Linking gate UI | `website/src/app/ui/components/meeting/MeetingLinkingScreen.tsx` — full-viewport PENDING / FAILED chrome (§5.3) |

`Layout` in `website/src/types/extends/global.ts` includes the `"MEETING"` member; `PageRouteState` includes `orgHostOnly?: boolean`.

### 5.1) Live meeting session (`MeetingLiveProvider` + `/meeting`)

Meeting realtime is a **layout-owned** session on namespace `/meeting`, independent of the shared boot socket (§4). Exactly one socket, one Yjs document, and one reactive store exist for the Meeting shell tree. Consumers under the layout must not open a second session.

Dependencies: `yjs@13.6.27`, `@syncedstore/core@0.6.0`, `@syncedstore/react@0.6.0` (all exact-pinned, `yjs` matched to the backend copy).

Module: `website/src/app/ui/components/meeting/hooks/useMeetingLive.tsx` (`.tsx` because it exports JSX for the provider). ThemeManager-shaped split:

| Symbol | Role |
|---|---|
| `useMeetingLiveInstance(data)` | Private session owner (socket + Yjs + SyncedStore). Called **only** inside `MeetingLiveProvider`. |
| `MeetingLiveProvider` | Reads `Meeting` route params via `useCurrentParams({ ident: "Meeting", mapParams: p => p })`, calls `useMeetingLiveInstance` once, publishes `MeetingLiveHookOp` on `MeetingLiveContext`. |
| `useMeetingLive()` | Public default export — `useContext(MeetingLiveContext)` only (no args). Empty default when used outside the provider (same shape as `useThemeManager`). |

`MeetingLayout` wraps both desktop and mobile shell trees in **one** outer `<MeetingLiveProvider>` so the transport session is not duplicated per breakpoint branch. `MeetingLiveSessionProvider` nests under it (product session — §5.3).

**Document bundle** (inside `useMeetingLiveInstance`). Shape is `{ [MEETING_LIVE_MAP]: Partial<MeetingLiveMap> }`. `createMeetingLiveDocBundle()` builds `syncedStore({ [MEETING_LIVE_MAP]: {} })` plus its `getYjsDoc(store)`:

- Root key is the mirrored `MEETING_LIVE_MAP` from `types/meeting.ts`, not a handwritten string.
- Root value **must** be an empty `{}`. Nesting e.g. `participants: {}` / `agendaItems: {}` in the SyncedStore initializer throws at runtime; the CRDT fills nested maps after sync.
- The public hook reads `liveStore[MEETING_LIVE_MAP]` (typed `Partial<MeetingLiveMap>`), including `participants` / `currentTalkMemberId` / `agendaItems` / `currentAgendaItemId` once the server document has them.

The bundle is held in a ref keyed by `meetingId` and rebuilt when that id changes, so a route param change cannot keep editing the previous meeting's document. `useSyncedStore(store, [store])` receives the store in its dependency list — without it the memoized proxy would keep pointing at the old document.

**Session binding** (`bindMeetingLiveSession`), after the CSR gate (`useClientPreparing`) and the `memberId` / `memberToken` / `meetingId` guard:

1. `createSocketInstance(getMeetingSocketInstanceConfig(...))` with the handshake query from `meeting-socket.ts` (§3).
2. `doc.on("update")` → skip anything whose origin is `"remote"`, otherwise convert with `Y.convertUpdateFormatV1ToV2` and emit `meeting.live.update`. Local edits are always emitted, including before the first sync.
3. `connect` → `connected = true` and emit `meeting.live.sync` with the local `Y.encodeStateVector`.
4. `connect_error` → clear `connected` / `synced`. Branch on Engine.IO error shape:
   - `err.type === "TransportError"` (network / websocket transport only) → leave `error` untouched so linking stays **PENDING**, and **do not** disable Socket.IO reconnection.
   - any other `connect_error` (including handshake `NOT_VALID_CREDENTIAL` from `meeting_auth`) → `error = "NOT_VALID"`, set `socket.io.opts.reconnection = false`, then `socket.disconnect()` so the UI settles on **FAILED** instead of bouncing in PENDING.
5. `meeting.live.sync` reply → apply `update` when present, then answer the server's `stateVector` with `Y.encodeStateAsUpdateV2(doc, vector)` so anything edited offline is pushed back; clear `error` and set `synced`.
6. `meeting.live.update` → apply with origin `"remote"` so the change is not echoed back.
7. `meeting.live.error` → store the code and clear `synced` (post-connect write/sync reject path; does **not** disable reconnection by itself).
8. `disconnect` → clear `connected` and `synced`; on `io server disconnect` call `socket.connect()` (Socket.IO does not auto-reconnect after a server-forced drop).
9. Cleanup on unmount / deps change: `doc.off`, unregister the three listeners, `off` native `connect` / `connect_error` / `disconnect`, `disconnect`, reset state.

Provider / public hook value: `{ connected, synced, error, meeting, batch }` (`MeetingLiveHookOp`). `meeting` is the reactive `Partial<MeetingLiveMap>` proxy; `batch(fn)` is `doc.transact(fn)` and is the transport write primitive. `MeetingLiveMap` (including `participants`, `currentTalkMemberId`, `agendaItems`, `currentAgendaItemId`) lives in the mirrored pair `website/src/types/meeting.ts` ↔ `backend/src/app/types/meeting.ts` with **no** GQL imports — see `.cursor/rules/meeting-live-map-mirror.mdc`.

Product UI should prefer `useMeetingLiveSession()` (§5.3) for `linking` / `can` / `actions` and for **reading** `meeting` / `me`. Keep `useMeetingLive()` for raw transport flags (`connected` / `synced` / `error`) and for `batch` **only when implementing a new session action**. Product screens and shell UI must not call `batch` or assign live fields directly.

Base64 helpers (`toBase64` / `fromBase64`) are local to the module because the payloads travel as base64 strings on the socket.

Backend pairing (`docs/platforms/backend/contracts/meeting-realtime-socket.md`): `meeting_auth` proves the handshake (refusal → Socket.IO `connect_error`, **not** `meeting.live.error`), the connection controller joins `Rooms.MEETING(meetingId)` and keeps both live events bound, and a rejected write answers `meeting.live.error` without unbinding anything.

**Not mirrored:** `meeting.live.*` are session events of this namespace. Do not add them to `website/src/types/events.ts` / socket event registries.

Authority: `.cursor/rules/meeting-realtime-socket.mdc`, `.cursor/rules/meeting-live-state.mdc`, `.cursor/rules/website-meeting-live-session.mdc`, `.cursor/rules/website-meeting-shell.mdc`, `.cursor/skills/meeting-realtime-socket/SKILL.md`, `.cursor/skills/website-meeting-live-session/SKILL.md`, `.cursor/skills/website-meeting-shell/SKILL.md`.

### 5.2) Current participant (`useMeetingLiveMe`)

File: `website/src/app/ui/components/meeting/hooks/useMeetingLiveMe.ts`.

Returns the **SyncedStore proxy** for the current member's entry in `meeting.participants`, keyed by the Meeting route `memberId` (`useCurrentParams` for identify `Meeting`).

| Concern | Behavior |
|---|---|
| Source | `useMeetingLive().meeting.participants[memberId]` |
| Return type | `MeetingLiveParticipant \| undefined` |
| Missing data | `undefined` when `memberId` is absent, `participants` is not synced yet, or that id is not in the map |
| Writes | Product UI must use session `actions` (§5.3). Action implementers mutate the proxy inside `useMeetingLive().batch(...)`. Do not assign fields from screens. |

**Must not clone.** Spreading, `Object.assign`, or any new object would break SyncedStore observation; `batch` edits would not reach the CRDT. Index and return the proxy as-is.

Naming: under the `MeetingLive*` family, `Me` means the session member (parallel to customer `useMe`), not a separate Member GQL load.

The Meeting page does not consume this hook directly; it is the index helper used inside `MeetingLiveSessionProvider` (§5.3) to populate session `me`. READY shell (`MeetingHeader` / `MeetingDrawerPanel`) reads `me` from `useMeetingLiveSession()`.

### 5.3) Meeting live session surface (`linking` / `can` / `actions` / `meeting` / `me` / `attendWindow`)

Public product API for Meeting UI. Transport stays in §5.1 (`MeetingLiveProvider`). Session is a **nested** provider under it: `MeetingLiveSessionProvider` owns one resolve + `actions` + **one** attend-window clock for the whole tree; consumers read via `useMeetingLiveSession()`.

| Piece | File | Role |
|---|---|---|
| Attend-window clock | `website/src/app/ui/components/meeting/hooks/useMeetingAttendWindow.ts` | Mirror of `MeetingModel.ATTEND_OPEN_BEFORE_MS` (math **private**). Returns `{ opensAtIso, windowOpen, nowMs }`. While waiting: one open `setTimeout` + 30s ticks. Invalid `datetime` → no `opensAtIso`, `windowOpen` false |
| Pure session state | `website/src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | `resolveMeetingLiveSession` → `MeetingLiveSessionState` (`linking` + `can`); takes `windowOpen` for `can.attend` |
| Provider + hook | same file | Calls `useMeetingAttendWindow(meeting.datetime)` **once**; passes `windowOpen` into resolve; exposes `attendWindow` on the session value |
| Gate UI | `website/src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | Full-viewport PENDING / FAILED when linking is not READY |

#### Public shape

```ts
{
  linking: MeetingLiveLinking;           // "PENDING" | "READY" | "FAILED"
  can: MeetingLiveCapabilities;
  actions: MeetingLiveSessionActions;
  meeting: Partial<MeetingLiveMap>;     // read
  me: MeetingLiveParticipant | undefined; // read
  attendWindow: MeetingAttendWindowHookOp; // opensAtIso / windowOpen / nowMs
}
```

`MeetingLiveLinking` is the **only** derived session enum. Meeting lifecycle and attendance are read from `meeting.status` and `me.attendedAt` / `me.leftAt` — no remapped stage enums.

**Single clock.** Product screens (including `MeetingInitPage`) must **not** call `useMeetingAttendWindow`. They read `attendWindow` from session. Do not re-export private mirror helpers (`ATTEND_OPEN_BEFORE_MS`, `isAttendWindowOpen`, `attendOpensAtIso`).

#### Write contract

| Surface | Role |
|---|---|
| `meeting` / `me` on session | **Read** reactive truth. Product screens must not assign fields. |
| `actions.*` | **Only** product write path (`can.*` then internal `batch`). |
| `useMeetingLive().batch` | Transport/CRDT hook only — never re-exported from session; never called from Meeting page/shell UI. New writes = new `actions` + `can` entry. |

Proxies remain technically mutable; enforcement is this contract plus governance rules/skills.

#### Linking (`MeetingLiveLinking`)

| `linking` | Condition |
|---|---|
| `FAILED` | `error != null` (from `meeting.live.error` **or** non-transport `connect_error`) |
| `READY` | `connected && synced` and no error |
| `PENDING` | otherwise (including transport-only `connect_error` while Socket.IO retries) |

When `linking !== "READY"`, every `can.*` = `false`. The session value still includes `meeting` / `me` / `attendWindow`; the shell does not mount product chrome until READY.

#### Capabilities (`can`)

Computed only when linking is READY. Inputs include live `status`, `meType`, `attendedAt`, `leftAt`, and `windowOpen` from session `attendWindow`.

| Key | True when |
|---|---|
| `startMeeting` | `meType === "CHAIRPERSON"` and `status === "WAITING_TO_START"` |
| `endMeeting` | chairperson and `status === "STARTED"` |
| `attend` | any `meType`, status `WAITING_TO_START` or `STARTED`, no `attendedAt`, no `leftAt`, and `windowOpen` (open at `datetime − MeetingModel.ATTEND_OPEN_BEFORE_MS`) |
| `left` | any `meType`, status `WAITING_TO_START` or `STARTED`, has `attendedAt`, no `leftAt` |
| `enterLive` | has `attendedAt` and no `leftAt` (any type) — **navigation gate only** (no `actions.enterLive` write) |
| `muteAllMedia` | chairperson and `status === "STARTED"` — **UI gate only** for the broadcast mute-all controls (no `actions` entry; the command rides the LiveKit data channel, `flow-meeting-broadcast.md` §7) |

Attend stays allowed after the window opens for the rest of a live session (`WAITING_TO_START` or `STARTED`). Missing / invalid `datetime` keeps `windowOpen` false → `can.attend` false.

These are **client session gates**. They do **not** replace the backend write gate (`MEETING_LIVE_STATUSES` on `meeting.live.update`), do **not** validate `attendedAt` timestamps on the socket, and do **not** yet enforce participant type on the server (§8).

**Clock refresh.** Session-owned `useMeetingAttendWindow` flips `windowOpen` with one `setTimeout` at the open instant (plus 30s ticks while waiting for Init remaining-duration copy). Background-tab throttling, effect cleanup on `datetime` changes, leaving the meeting route, and large forward clock jumps while a timeout is pending can delay or cancel that flip until the effect runs again.

#### Actions (sync)

Built only in `MeetingLiveSessionProvider` / `useMeetingLiveSessionInstance` (need `batch` + proxies). Each action no-ops when the matching `can.*` is false (and `attend` / `left` also require `me`):

| Action | Write inside `batch` |
|---|---|
| `startMeeting` | `meeting.status = "STARTED"` |
| `endMeeting` | `meeting.status = "COMPLETED"` |
| `attend` | `me.attendedAt = new Date().toISOString()` |
| `left` | `me.leftAt = new Date().toISOString()` |

Actions are synchronous (`() => void`). Socket fan-out is side effect of the CRDT update path; a later `meeting.live.error` sets `error` and clears `synced` (§5.1).

#### Linking gate in `MeetingLayout`

`MeetingShell` (inside `MeetingLiveSessionProvider`) calls `useMeetingLiveSession()` and, when `linking !== "READY"`, returns **only** `<MeetingLinkingScreen />` — no header, drawer, footer, or page children.

`MeetingLinkingScreen` reads `linking` from `useMeetingLiveSession()` (org colors via `useOrganization`):

| Linking | Chrome |
|---|---|
| `PENDING` | Org logo and/or name (both when present), project `Loadable` (`loading`), `pendingStatus` copy |
| `FAILED` | Early return: shared `LaneFailed` (`components/Wrong`) with `failedMessage` title + `failedHint` description, inside a `relative` full-height wrapper on org `pageBackground`. No org identity block on this path, and no retry button (there is no retry entry point) |

`LaneFailed` is an absolute fill, so the wrapper must be `relative`; do not hand-roll an icon + message pair here again (`flow-meeting-broadcast.md` §8).

i18n under `ui.layouts.meetingLayout.linking`: `logoAria`, `pendingStatus`, `failedMessage` (card title — no trailing period), `failedHint` (ar + en).

### 5.4) Meeting shell (organization branding)

`MEETING` is the only layout that renders tenant branding. The branding source is the `organizationHost` slice (§3) — the same public payload `org/start` returns — so the shell needs no extra request.

#### `useOrganization`

`website/src/app/ui/components/meeting/hooks/useOrganization.ts` is a product hook under `components/meeting/hooks/`, not `ui/base/hooks` (same placement rule as `useMeetingLive`).

- Reads `state.organizationHost` through `useSector` and returns `null` when `id` is absent, so a consumer cannot render tenant chrome on a non-organization host.
- Returns `{ id, name, description, logo_url, colors }`. Raw `primary_color` / `secondary_color` are **not** re-exported — a consumer must go through `colors`.
- `colors` is an `OrganizationColors` map whose keys mirror `semanticColor` naming for the shell half, plus computed brand and scheme-pair keys. A consumer reads `colors.cardBackground` like `semanticColor.cardBackground`, and attendance tiles use dedicated `idleCardBackground` / `presentCardBackground` — never invent a local fill when contrast fails.
- Seeds resolve with the installed `color` package: `primary_color` → brand, `secondary_color` → accent. A `null`, empty, or unparseable value falls back to `BrandColors.navy` / `BrandColors.orange`. The whole map is `useMemo`-ed on the six store fields. Light-scheme section tints (`sectionBrandBackground` / `sectionAccentBackground`) mix the seed toward white at **0.78** (not near-white wash) so chips and soft fills stay readable on `pageBackground`; dark-scheme section tints keep the existing soft rgba overlays. `presentCardBackground` is a stronger accent wash (**0.62** / **0.18**) so it stays distinct from type chips on `sectionAccentBackground`.
- `defaultOrganizationColors()` builds the same map from the two `BrandColors` fallbacks; every consumer uses `organization?.colors ?? defaultOrganizationColors()` so the shell renders before the slice hydrates.

**Token split (non-negotiable).** Shell/neutral keys are assigned the `semanticColor.*` token itself, never a copied literal — `ColorType` accepts a `ThemeMapPath` and `getColor` resolves it per scheme, so these follow `theme.ts` automatically. Brand keys are computed from the seeds. Scheme-pair keys are explicit light/dark `ColorType` pairs for surfaces that must diverge by scheme (or must not collide with chip fills):

| Group | Keys | Source |
|---|---|---|
| Shell (16) | `pageBackground`, `headerBackground`, `footerBackground`, `drawerBackground`, `cardBackground`, `inputBackground`, `inputBorder`, `navigationBorder`, `footerBorder`, `subtleDivider`, `backdrop`, `textPrimary`, `textSecondary`, `textTertiary`, `iconPrimary`, `iconSecondary` | `semanticColor.<same key>` |
| Brand (9) | `primaryActionBackground`, `accentActionBackground`, `primaryActionText`, `accentActionText`, `actionIconOnFill`, `sectionBrandBackground`, `sectionAccentBackground`, `textBrand`, `textAccent` | computed from the org seeds (`actionIconOnFill` is fixed white for solid action wells) |
| Scheme pair (2) | `idleCardBackground`, `presentCardBackground` | idle: light `#FFFFFF` / dark `@transparent`; present: stronger accent wash than `sectionAccentBackground` (softLight **0.62** / softDark **0.18**) so type chips stay distinct |

**Rule:** need a new fill? Add a token here — do not branch `colorScheme` in the component and do not reuse a chip fill for a card body.

Copying a `ThemeMap` leaf into the shell group is a defect: `yarn type-check` validates a token path but cannot detect a stale hex, so a copy drifts silently on the next `theme.ts` edit. `primaryActionText` / `accentActionText` pick white or ink by seed luminance, so an organization may choose a light primary without losing contrast. Authority: `docs/invariants/website.md` W43.

`sectionBrandBackground` / `sectionAccentBackground` / `textBrand` / `textAccent` / `accentActionText` are consumed by in-shell surfaces (e.g. `MeetingInitPage` meta chips and attended strip; selected drawer tiles use soft `sectionAccentBackground` + `textAccent` + partial start rail). The YAGNI clause in W43 applies to `semanticColor` additions, not to this map's brand half, which is generated from the same two seeds regardless.

#### Shell components

| Component | Shipped behavior |
|---|---|
| `meeting/MeetingHeader.tsx` | Menu button (shared `HeaderIconButton`) + org logo, or the org name clamped to one line when there is no logo; when `me` exists, trailing `MeetingHeaderMe` (avatar + name + type chip); for non-chair `me`, a disabled request-to-speak control (mic + `requestTalk` / `requestTalkAria`). A 2px rail in `colors.accentActionBackground` sits on the bottom edge. `fixed` prop switches between the in-flow desktop bar and the mobile `Fixed` bar at `zIndex.header`. |
| `meeting/MeetingHeaderMe.tsx` | Presentational current-participant cluster: props `name` / `type` / `avatarUrl` / `colors` / optional `pushTrailing`. Avatar circle (~`2.2rem`) uses org `primaryActionBackground` + image or `FiUser` on `actionIconOnFill` (do **not** import `customer/IdentityAvatar`). Name: `name.trim()`, `smallAction` bold, ellipsis, `maxW={14}` on the cluster. Type chip: soft `sectionAccentBackground` + `textAccent`; labels `typeChairperson` / `typeMember` / `typeViewer` (`CHAIRPERSON` / `VIEWER` / else → member). Visible text only — no redundant group `aria-label`. |
| `meeting/MeetingFooter.tsx` | Rights line only: `© <year> <platform name> — <rights>`. The name is the **platform** (`ui.layouts.mainLayout.footerTitle`), not the organization. |
| `meeting/MeetingDrawerPanel.tsx` | Shared panel body for both breakpoints: title row (close when `showClose`), org identity card (logo and/or name), role-based 2-column tile grid with full-row `live` (Meeting room / غرفة الاجتماع) then full-row Meeting info (`itemHome` + `HomeMark` → `"init"`), then remaining role tiles; calls `setPage` from `useMeetingPage`; pinned appearance/language row (`ThemeModeSwitch` + `LanguageSwitch`, both `compact`). |
| `meeting/MeetingDrawerOverlay.tsx` | Mobile portal overlay following the `CustomerDrawer` pattern: `createPortal` to `body`, `zIndex.modals`, blurred backdrop, RTL-aware slide-in, `no-scroll-drawer` body lock, 220ms unmount delay. Exported **without** `withMemo` — it reads `router.isRTL`. |

Header and footer put their chrome inside `Container`. READY route children mount in the same width via `FlexContainer` (`dir="column"`, `flx_1`, `minH={0}`) in `MeetingLayout` — page / broadcast / overlay share that column. In-shell page bodies use vertical pad only (`pv`). `MeetingPageOverlay` adds horizontal pad equal to that token (`ph={page.padY}`) so floating pages keep equal inset on all sides.

The button chrome is `website/src/app/ui/components/HeaderIconButton.tsx` — a top-level shared component (`component-structure.md` §3), consumed by both `CustomerHeader` and `MeetingHeader`. It owns its own tokens (`inputBackground`, `inputBorder`, `iconPrimary`, `semanticDims.control.iconButtonSize`, `semanticDims.card.radius`) and exposes no color props, so the two headers cannot drift.

The glyph is the shared `DrawerMenuIcon`. Its leading bar is themeable through the optional `accentClr` prop, defaulting to `semanticColor.accentActionBackground`; the two horizontal bars are always `semanticColor.iconPrimary`. `MeetingHeader` passes `semanticColor.iconPrimary` so all three bars read as one content-colored mark (dark in light scheme, near-white in dark). `CustomerHeader` passes nothing and keeps the accent bar.

**Drawer tiles.** Role-based from `useMeetingLiveSession().me?.type` (local `DrawerGridItemDef[]` with `id: MeetingDrawerPage` = `Exclude<MeetingPage, "init">` in `MeetingDrawerPanel`). The READY shell (header/drawer) mounts only after `linking === "READY"`. Order in the 2-column grid: full-row `live` first (Meeting room / غرفة الاجتماع), then full-row **Meeting info** (`itemHome` + shared `HomeMark` forced white via `actionIconOnFill` on both `frameClr` and `accentClr` — same monochrome-on-fill idea as `DrawerMenuIcon accentClr` on the menu button; do not use `FiHome`) calling `setPage("init")` (selected when `page === "init"`; label Meeting info / معلومات الاجتماع), then the remaining role tiles. Chairperson (`CHAIRPERSON`): `live` (`FiVideo`), Meeting info, `talkQueue` (`FiMic`), `attendance` (`FiClipboard`), `agenda` (`FiList`), `decisionsAndVote` (`FiCheckSquare`). Member and viewer: `live`, Meeting info, `agenda`, `decisionsAndVote` only. No `participants` tile and no `init` id inside `DrawerGridItemDef[]`. The `live` and Meeting info tiles are full-row (`wide` → `gridColumn: 1 / -1`); other tiles stay single-cell. Tiles call `useMeetingPage().setPage(id)` (and `onClose` on the mobile overlay). When `!can.enterLive`, the `live` tile is **disabled** (`opc` 0.55, no hover scale, native `disabled` / `aria-disabled`); visible label stays Meeting room / غرفة الاجتماع; `title` / `aria-label` use `init.attendRequiresForRoom`. While `meeting.status === "STARTED"` and `can.enterLive`, the `live` tile shows a corner accent broadcast ping (`accentActionBackground` pulse; a11y label appends `init.statusStarted`). Other drawer pages are not gated on attendance.

**Selected tile chrome** (landing FAQ / message-card family — not solid primary fill):

| Trait | Shipped value |
|---|---|
| Soft fill | `sectionAccentBackground` (secondary seed soft tint) when selected; otherwise `cardBackground` |
| Start rail | When selected: absolute `3px` `accentActionBackground` bar, `insetInlineStart: 0`, top/bottom inset `0.85rem` (partial height, not full card) |
| Label | `textAccent` when selected; otherwise `textPrimary` |
| Icon well | Always `primaryActionBackground` + white `actionIconOnFill` glyph (or monochrome white `HomeMark`); **no** selected accent swap |
| Perimeter border | Always `inputBorder` — **no** full selected brand border |
| A11y | `aria-current="page"` when selected |

Tile layout mirrors `CustomerDrawer`'s `DrawerGridItem`: `minH 7.2`, a `3.1`-square icon well, and a `smallAction` bold label clamped to two lines. `DrawerGridItem` accepts either `Icon` (Feather) or `mark` (ReactNode) — Home uses `mark`. Governance: `.cursor/rules/website-meeting-shell.mdc`, skill `website-meeting-shell`.

**Header identity.** When `me` exists, `MeetingHeader` mounts `MeetingHeaderMe` on the trailing side with `pushTrailing` (cluster owns `ml="auto"`). Props come from live session `me` (`name`, `type`, `avatarUrl`) plus org `colors` — no extra GQL. Request-to-speak sits **after** the cluster and takes `ml="auto"` only when `me` is absent. Do not import `customer/IdentityAvatar` (portal `semanticColor`); avatar well ink is org `actionIconOnFill`. Type labels: `header.typeChairperson` / `typeMember` / `typeViewer`. Name display is `name.trim()` (no invented empty-name glyph). Cluster width capped with numeric rem shorthand `maxW={14}`.

**Header request-to-speak.** When `me` exists and `me.type !== "CHAIRPERSON"`, `MeetingHeader` shows a disabled control after the identity cluster: mic + visible `requestTalk` label; accessible name `requestTalkAria` (session-scoped, distinct from the label). MEMBER and VIEWER both get the control (`meeting-participant-domain.md` §8: request talking for all three types; chairperson uses drawer `talkQueue` instead). Shipped chrome is icon + label only — no switch track, no click handler. Header reads `me` from `useMeetingLiveSession()`.

#### Layout composition

`MeetingLayout` wraps both breakpoint trees in a single outer provider chain: `MeetingLiveProvider` → `MeetingLiveSessionProvider` → `MeetingPageProvider`. Inside, `MeetingShell` runs the linking gate (§5.3) **before** picking desktop vs mobile chrome: when `linking !== "READY"`, only `MeetingLinkingScreen` mounts (page `children` are not rendered). When READY, the shell picks one of two trees from a `matchMedia(min-width: SW.min_lg)` effect (same shape as `MainLayout`). `children` is rendered once per READY tree.

- **Desktop:** a `Row` of [drawer column | `Col`(header, `FlexContainer` content, footer)]. The drawer column is **in flow** (not an overlay) and `position: sticky; top: 0` at `h/maxH: 100vh`, so it stays viewport-tall while the page scrolls and never covers the footer. Its width animates between `semanticDims.shell.drawerWidth` and `0`, with `pointerEvents: none` while collapsed. Content children sit in `FlexContainer` (`dir="column"`, `flx_1`, `minH={0}`) — same width contract as header/footer `Container` (`defaultContainerSizeDims`).
- **Mobile:** the `CustomerMainLayout` shape — `MeetingHeader fixed`, content `minH: 100vh` with a `paddingTop` matching `Dims.headerHeight` / `Dims.mobileHeaderHeight`, then the same `FlexContainer` for children, and the portal overlay.

The panel is capped at `maxH: 100vh` with `minH: 0`; only the tile grid scrolls (`Col` + `flx_1` + `minH={0}` + `customScroll`), which keeps the appearance/language row visible at the bottom instead of pushing it below the fold as the tile list grows.

**i18n.** `ui.layouts.meetingLayout` in both `ar.ts` and `en.ts`, with identical key sets:

| Group | Keys |
|---|---|
| `header` | `menu`, `logoAria`, `typeChairperson`, `typeMember`, `typeViewer`, `requestTalk`, `requestTalkAria` |
| `footer` | `rights` |
| `linking` | `logoAria`, `pendingStatus`, `failedMessage`, `failedHint` |
| `drawer` | `title`, `closeAriaLabel`, `logoAria`, `itemHome`, `itemLive`, `itemTalkQueue`, `itemAttendance`, `itemAgenda`, `itemDecisionsAndVote`, `utilityPrefs` |
| `init` | `logoAria`, type/status labels, `attend` / `attendAria`, `attendAvailableIn`, `attendRequiresForRoom`, `attendedTitle` / `attendedAt`, `roomUnlockedHint`, `leftTitle` |
| `live` | `statusWaitingToStart`, `waitingLead`, `startMeeting` / `startMeetingAria` |
| `broadcast` | `connecting`, `connectionError` / `connectionErrorHint`, `videoOn` / `videoOff` / `videoAria`, `micOn` / `micOff` / `micAria`, `soundOn` / `soundOff` / `soundAria`, `muteAllVideos` / `muteAllVideosAria`, `muteAllMics` / `muteAllMicsAria` (`flow-meeting-broadcast.md` §9) |
| `overlay` | `closeAria` |

`MeetingFooter` additionally reads `ui.layouts.mainLayout.footerTitle` for the platform name; it has no key of its own for it.

### 5.5) In-shell page (`MeetingPage`)

One `Meeting` **route** (`ui/pages/Meeting.tsx`). In-shell navigation is **local React state**, not URL segments and not a second route registry entry.

**Provider placement.** `MeetingPageProvider` sits in `MeetingLayout` under live + session providers (same chain style — not inside `MeetingShell` after READY):

`MeetingLiveProvider` → `MeetingLiveSessionProvider` → `MeetingPageProvider` → `MeetingShell`

Because the provider wraps `MeetingShell`, `page` state survives linking PENDING/FAILED ↔ READY transitions for the life of the layout mount. During non-READY, shell content (drawer + route children) does not mount, so nothing reads `page` until READY.

| Piece | Path | Role |
|---|---|---|
| `MeetingPage` | `hooks/useMeetingPage.tsx` | `"init" \| "live" \| "talkQueue" \| "attendance" \| "agenda" \| "decisionsAndVote"` |
| `MeetingPageProvider` / `useMeetingPage` | same file | `useState<MeetingPage>("init")`; exposes `{ page, setPage }` |
| `MeetingDrawerPage` | local type in `MeetingDrawerPanel.tsx` | `Exclude<MeetingPage, "init">` — tile ids only |
| Page switch | `ui/pages/Meeting.tsx` (`MeetingContent`) | `switch (page)` — owns which body mounts |
| Page bodies | `components/meeting/pages/*` | Product UI per id; not under `ui/pages/` (route entries stay there) |

**Default.** `"init"` → `meeting/pages/MeetingInitPage.tsx` — shared lobby for every participant type (`CHAIRPERSON`, `MEMBER`, `VIEWER`):

| Block | Behavior |
|---|---|
| Org identity | Large logo (`h: 6rem`) or primary monogram (first letter on `primaryActionBackground` / `primaryActionText`) when `logo_url` is absent; org name (`subHead`) |
| Meeting meta | `meeting.subject` when present; type/status chips via shared `MeetingMetaChip` (`tone="brand"` for type, `tone="accent"` for status; org section fills). Missing live fields omit their UI (no placeholders). Not the customer-portal `MeetingMetaChips` (plural) component |
| Attendance | Owned by colocated `InitAttendSection` inside `MeetingInitPage.tsx` (page-only — not a shared file, not a `render*` helper). Branch order: (1) `can.attend` → `MeetingPrimaryButton` → `actions.attend` + caption `attendRequiresForRoom` under the button. (2) Present (`can.enterLive`) → confirmation strip (check + first-person `attendedTitle` + first-person relative `attendedAt` via `moment(attendedAt).from(now, true)`) + quieter `roomUnlockedHint` (`caption` + `textTertiary`). (3) `leftAt` → `leftTitle`. (4) Else → accent info strip: when `attendWindow` says the open window has not started yet, `attendAvailableIn` with Moment locale-aware remaining duration (`moment(opensAtIso).from(moment(nowMs), true)`, refreshed by session clock every 30s while waiting) + `attendRequiresForRoom`. If the window is already open but `can.attend` is still false (e.g. non-live status), show `attendRequiresForRoom` only — never a past “available in” duration. No disabled fake attend button. Section reads `attendWindow` from session — does not call `useMeetingAttendWindow` |
| Colors / copy | `organization?.colors ?? defaultOrganizationColors()` only. Copy under `ui.layouts.meetingLayout.init` (ar + en, identical keys). Init attend strings are **first person**; roster strings under `meetingLayout.attendance` are third person |

**Meeting room (`live`) gate.** All participant types need present check-in (`can.enterLive`) before the Meeting room page. Drawer: disabled `live` tile when locked (§5.4). `MeetingLivePage` also redirects to `"init"` when `!can.enterLive` (defense if `page` was already `"live"`). Agenda / talkQueue / other drawer ids stay ungated. Chair `startMeeting` / `endMeeting` are unchanged.

**Meeting room (`live`) body.** `MeetingLivePage` is waiting/start only (after the enterLive gate):

| Condition | UI |
|---|---|
| Waiting (typically `WAITING_TO_START`) | Centered waiting copy (`statusWaitingToStart` + `waitingLead`); when `can.startMeeting` show `MeetingPrimaryButton` → `actions.startMeeting` |
| `STARTED` | Not rendered as exclusive page content — see persistent broadcast stack below |

**Persistent broadcast + floating pages (`STARTED`).** Owned by `Meeting.tsx` `MeetingContent`:

| Condition | UI |
|---|---|
| `status !== "STARTED"` | Exclusive `renderPage(page)` (same as before) |
| `STARTED` and `page === "live"` | `MeetingLiveBroadcast` only — real LiveKit A/V stage |
| `STARTED` and `page !== "live"` | `MeetingLiveBroadcast` stays mounted underneath; current page in `MeetingPageOverlay` |

**Broadcast body (LiveKit).** `MeetingLiveBroadcast` owns the media surface through `useMeetingLiveKitRoom` (token → `Room.connect` → peers → publish/playback toggles → cooperative mute-all). Because `Meeting.tsx` keeps it mounted for the whole `STARTED` span, the room survives in-shell page switches; leaving `STARTED` or unmounting disconnects it. Stage states are exclusive (media stack / `Loadable` / `LaneFailed`), controls are camera + mic + **sound** (playback is a separate control from the mic), and the chair mute-all group renders on `can.muteAllMedia`. Full contract, ceilings, and failure modes: `flow-meeting-broadcast.md`.

**`MeetingPageOverlay` contract (observed):**

| Trait | Shipped value |
|---|---|
| Geometry | Absolute fill of the READY content `FlexContainer`; solid sheet `l`/`r` `0` (same column width as header/footer `Container`); vertical float inset `t`/`b` = `page.padY` |
| Surface | Org `pageBackground` + `inputBorder` + `card.radius`; **solid only** — no blur, frost, scrim, or transparency experiments |
| Scroll body | `ph={page.padY}` so floating page content matches page bodies’ `pv={page.padY}` (equal inset on all sides while floating) |
| Close | Floating over content (compact theme/lang chrome: `semanticColor.input*` + size `2.35`); `overlay.closeAria`; **no** reserved top pad that defeats floating |
| Dismiss | Close control, outside tap (vertical margin / backdrop Absolute), or Meeting room drawer tile → `setPage("live")` |

`STARTED && page === "live" && !can.enterLive` → effect `setPage("init")`. Drawer `live` tile shows a corner accent ping while `STARTED` and enterLive.

Member / viewer see the same waiting panel without the start button (`can.startMeeting` false). Copy under `ui.layouts.meetingLayout.live` (ar + en). Shared org chrome: `MeetingPrimaryButton` and `MeetingMetaChip`. Status / type **enum display strings** must match backend `enums.meetingStatus` / `meetingType` labels in `backend/src/resources/trans/{ar,en}/general.ts` (e.g. `STARTED` = بدأ / Started).

**Drawer `live` tile label.** Tile id stays `"live"`; user-facing copy is **Meeting room** (en) / **غرفة الاجتماع** (ar) via `meetingLayout.drawer.itemLive` — not “Live” / “البث”.

**Drawer pages.** Each drawer id mounts a named page under `meeting/pages/`. **Attendance** (`MeetingAttendancePage`) and **Meeting room** (`MeetingLivePage` waiting/start) are shipped product UI. Other drawer ids still use shared `MeetingPageStub` until their product UI ships.

| `MeetingPage` id | Component | Shipped body |
|---|---|---|
| `live` | `MeetingLivePage` | waiting panel + chair start; bounce to `"init"` when `!can.enterLive` (broadcast owned by `Meeting.tsx` when `STARTED`) |
| `talkQueue` | `MeetingTalkQueuePage` | stub |
| `attendance` | `MeetingAttendancePage` | chair attendance log (below) |
| `agenda` | `MeetingAgendaPage` | stub |
| `decisionsAndVote` | `MeetingDecisionsAndVotePage` | stub |

Shared placeholder: `meeting/pages/MeetingPageStub.tsx`. Replace stub body inside the matching named page file when product UI ships — keep the file; do not invent a second route.

**Attendance log (`"attendance"`).** Chairperson-only. `MeetingAttendancePage` bounces non-chair with `useEffect` → `setPage("init")` and render `null` (same pattern as locked `MeetingLivePage`). Data from `useMeetingAttendance` (no navigation side effects). Reads roster from session `meeting.participants` only (no GQL, no direct `batch`).

| Block | Behavior |
|---|---|
| Title / subtitle | `attendance.title` + `attendance.subtitle` (page uses only the `attendance` translator — not `drawer`) |
| Quorum strip | Shown only when `typeof meeting.minMembersCount === "number"` and `minMembersCount > 0`. Present count = `countQuorumPresent` (`CHAIRPERSON` \| `MEMBER` + bucket `present`). Copy: `quorumLabel` / `quorumValue` (`:present` / `:required`) / `quorumMet` \| `quorumUnmet`. Hide strip when `minMembersCount` is missing (pre-seed BLOBs) — do not invent `0` |
| Filters | Shared `FilterCountChips` (`ui/components/`): page passes resolved `options` (label+count) + `value` / `onChange`; optional org `colors` chrome. No meeting translator inside the chip group |
| Grid | Utils `Grid` `cols` `{ default: 3, [SW.max_md]: 2, [SW.max_sm]: 1 }`. Empty participants → `Empty` (`emptyParticipants` + `subtitle`). Empty filter → `Empty` (`emptyFilter` only) |
| Card | `MeetingAttendanceCard`: own avatar/name/type chrome (visual language shared with header identity; **not** `MeetingHeaderMe`). `statusLabel` from data hook. `present` prop. Present: `presentCardBackground` + check disc. Type chip: `sectionAccentBackground`. Else: `idleCardBackground` + `subtleDivider` |
| Sort | `sortAttendanceParticipants`: present → awaiting → left |
| Copy | `ui.layouts.meetingLayout.attendance` (ar + en). `filterPresent` = سجّل / Checked in |

Helpers: `meeting/hooks/useMeetingAttendance.ts` (pure helpers + filter / quorum / rows; no navigation). Filters UI: shared `FilterCountChips`.

**Drawer writes.** Tiles call `setPage(id)` and, on the mobile overlay, `onClose`. Meeting info (`itemHome` + `HomeMark`) is full-row **after** `live` inside the grid (`setPage("init")`; selected when `page === "init"`; label Meeting info / معلومات الاجتماع). Selected tile chrome: soft `sectionAccentBackground`, partial-height start accent rail (`3px`, inset), `textAccent` label; icon well stays `primaryActionBackground` + white glyph; `aria-current="page"`. While `STARTED` and enterLive, the `live` tile shows a corner accent broadcast ping.

Governance: `.cursor/rules/website-meeting-shell.mdc`, skill `website-meeting-shell`. Authority for shell chrome and org color map remains §5.4.

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

**Wired on** `POST /custom/org/livekit_token` (per-route). `/custom/org/start` does not use it — see §8 and `../backend/contracts/livekit-media-plane.md` §6.

### 6.3 Socket `/meeting`

Full backend contract: `docs/platforms/backend/contracts/meeting-realtime-socket.md`.

What the website side depends on:

- namespace `/meeting`, registered in `backend/src/resources/configs/socket/io.ts` with `globalMiddlewares: ["meeting_auth"]`,
- `MeetingAuthenticationIOMiddleware` proves member token + meeting + roster row + `ACTIVE` organization before any controller runs, so a refused handshake surfaces as a connect error, not a partial session,
- `MeetingConnectionIOController` joins `Rooms.MEETING(meetingId)` and binds `["meeting.live.sync", "meeting.live.update"]`,
- the live controllers keep that set bound on every reply, including rejections, so the client only has to re-run the sync handshake on `connect` (§5.1),
- writes are refused unless the meeting status is `WAITING_TO_START` or `STARTED`, answered as `meeting.live.error` with code `MEETING_NOT_LIVE`.

## 7) Failure modes

| Scenario | Behavior |
|---|---|
| Empty/unparsable `Host` on SSR | `resolveRequestHost` throws `"unexpected"` → server errors funnel |
| Organization host with unknown subdomain/custom domain | `org/start` returns `404` → errors funnel redirects to `Error` with `404` |
| Production `org/start` body without `subdomain` and without `customDomain` | `NOT_PERMIT` → `Error` with `403` |
| Non-production without `TEST_ORGANIZATION_ID` | `org/start` `404` |
| Apex request to `/meeting/...` | route gate → `Error` `404` |
| Organization-host request to any non-`orgHostOnly` route (`/`, `/login`, `/customer/*`) | route gate → `Error` `404` |
| Socket `/meeting` handshake missing ids, bad token, org mismatch, or no roster row | Server throws `NOT_VALID_CREDENTIAL` → Socket.IO `connect_error` (not `meeting.live.error`) → client sets `error = "NOT_VALID"`, stops reconnection → `linking === "FAILED"` (§5.1, §5.3) |
| Network / websocket transport failure before connect | `connect_error` with `type === "TransportError"` → stay PENDING; Socket.IO keeps retrying (§5.1) |
| Live write on a meeting that is not `WAITING_TO_START` / `STARTED` | `meeting.live.error` `MEETING_NOT_LIVE` → client `error` is set and `synced` clears → linking FAILED until a later successful sync clears `error` (§5.1) |
| Malformed live payload | `meeting.live.error` `NOT_VALID` → same linking FAILED path |
| Route params change to another meeting | `MeetingLiveProvider` re-runs `useMeetingLiveInstance` with the new ids; the instance drops the old document and store and opens a fresh session (§5.1) |

## 8) Known limits (shipped state, intentional)

1. **`org_host` is wired on LiveKit token fetch.** `POST /custom/org/livekit_token` uses per-route `middleware("org_host")`. `/custom/org/start` still resolves the organization from the body without `org_host`. The response now carries `{ token, url }`, and `useMeetingLiveKitToken` feeds `useMeetingLiveKitRoom` (no probe UI). Contract: `../backend/contracts/livekit-media-plane.md` §6.
2. **Most drawer in-shell pages are still title stubs (§5.5).** `TalkQueue` / `Agenda` / `DecisionsAndVote` mount and show the drawer label only (`MeetingPageStub`) until product UI is designed. **`MeetingAttendancePage` is shipped** (chair-only attendance log + quorum). **`MeetingLivePage` is shipped** (waiting + chair start). While `STARTED`, broadcast is owned by `Meeting.tsx` (`MeetingLiveBroadcast` + solid `MeetingPageOverlay` for other pages) and now carries real LiveKit A/V. Post-start side effects beyond the existing status write remain deferred. Header request-to-speak for MEMBER and VIEWER remains disabled. Header identity (`MeetingHeaderMe`) is shipped and live from session `me`.
3. **Collaborative live map fields** — `subject`, `type`, `status`, `datetime` (seeded scheduled start; not a collaborative edit target), `minMembersCount` (seeded quorum denominator on first empty BLOB only; not a collaborative edit target; missing on older BLOBs → clients hide quorum UI), `participants` (incl. session-only `talkTurn` default `null`), `currentTalkMemberId` (seeded `null`; who is speaking; set after `participants`), `agendaItems` (SQL line mirror + session-only per-item `status` default `WAITING` including `CANCELED` for in-session cancel, `isLiveCreated` / `isLiveUpdated` default `false`; no live delete), `currentAgendaItemId` (seeded `null`; session-only). Agenda / talk-queue writers are not shipped yet. Everything else on a meeting still goes through the customer GQL/requester path, and the live values are not reflected back onto the SQL columns yet (`../backend/contracts/meeting-live-state.md` §6).
4. **Non-production organization resolution ignores the request body** and always uses `TEST_ORGANIZATION_ID`, so local runs exercise a single organization.
5. **Handshake values travel primarily on the Socket.IO query** built in `meeting-socket.ts`. Header names are also read on the server (`headers.x || query.x`), but Node lowercases headers; do not rely on camelCase header-only delivery.
6. **`meeting.live.*` is not mirrored** into frontend event registries — it is a namespace session protocol. Outbound meeting notify events, when added, still follow `socket-event-mirroring.md`.
7. **Organization-host pages other than `Meeting` have no realtime channel.** A tenant-wide org socket is not part of the shipped surface.
8. **Server still has no participant-type write gate.** Any authenticated roster member can push live updates while status is live (`../backend/contracts/meeting-realtime-socket.md` §4). Website `can.startMeeting` / `can.endMeeting` / `can.attend` / `can.enterLive` are **client** gates on `useMeetingLiveSession`; the attend open window and Meeting-room attendance requirement are **not** enforced by socket controllers or SQL writers yet.
9. **The meeting shell picks its READY tree in JavaScript**, so SSR always emits the mobile tree and a desktop client swaps after hydration. While linking is not READY, only the gate screen renders (no drawer SSR flicker for that path). `drawerOpen` is one state shared by both breakpoints and is re-seeded on every breakpoint crossing, so a manually collapsed desktop drawer reopens after a resize across `SW.min_lg`.
10. **Non-transport `connect_error` maps to session code `"NOT_VALID"`** for UI purposes; the linking FAILED copy stays the generic `failedMessage` (no distinct credential string on screen yet).
11. **Broadcast mute-all is cooperative, not enforced.** The chair button is gated by `can.muteAllMedia`, but the receiving hook applies any well-formed data-channel command from any peer, and a muted peer can re-enable immediately. There is no `roomAdmin` server mute and no participant-type publish grant. Accepted for now; full ceiling list in `flow-meeting-broadcast.md` §10.
12. **A failed broadcast connection has no in-UI retry.** `Disconnected` or a rejected token renders the failure card and waits for a page reload; only network-level token failures retry quietly (`flow-meeting-broadcast.md` §10.3).
13. **Attend-window clock refresh is best-effort.** The session-owned `setTimeout` / 30s ticks in `useMeetingAttendWindow` can be delayed or cleared by background-tab throttling, effect dependency changes, leaving the meeting route, or large forward clock jumps while a timeout is pending (§5.3).

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
| `src/app/ui/pages/Meeting.tsx` | page switch; while `STARTED` persistent `MeetingLiveBroadcast` + `MeetingPageOverlay` for non-`live` pages | §5, §5.5 |
| `src/app/ui/components/meeting/MeetingPageOverlay.tsx` | solid floating sheet for non-`live` while `STARTED` (column width; `ph=page.padY`; floating close) | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingLivePage.tsx` | `"live"` waiting + chair start; bounce to `"init"` when `!can.enterLive`; `pv` only | §5.5, §8, §10j, §10l |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | persistent `STARTED` broadcast — LiveKit stage (featured chair + remote grid, track attach, camera/mic/sound controls, chair mute-all on `can.muteAllMedia`; `pv` only) | §5.5, §10j, §10l, §10m; `flow-meeting-broadcast.md` §6 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | `"init"` lobby; colocated `InitAttendSection`; meta via `MeetingMetaChip`; `pv` only | §5.5, §10j, §10k, §10l |
| `src/app/ui/components/meeting/pages/MeetingPageStub.tsx` | shared title-only placeholder chrome for non-init drawer pages; `pv` only | §5.5, §8, §10l |
| `src/app/ui/components/meeting/MeetingPrimaryButton.tsx` | org primary CTA (`label` / `ariaLabel` / `onClick`); used by Init attend + Live start | §5.5, §10j |
| `src/app/ui/components/meeting/MeetingMetaChip.tsx` | org type/status chip (`label` + `tone`); used by Init meta (not customer `MeetingMetaChips`) | §5.5, §10k |
| `src/app/ui/components/meeting/pages/MeetingTalkQueuePage.tsx` | `"talkQueue"` body (title stub) | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingAttendancePage.tsx` | `"attendance"` body — chair attendance log + quorum; `pv` only | §5.5, §8, §10f, §10l |
| `src/app/ui/components/meeting/pages/MeetingAgendaPage.tsx` | `"agenda"` body (title stub) | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingDecisionsAndVotePage.tsx` | `"decisionsAndVote"` body (title stub) | §5.5, §8 |
| `src/resources/translations/ar.ts`, `src/resources/translations/en.ts` | `ui.layouts.meetingLayout` (`header`, `footer`, `linking` incl. `failedHint`, `drawer` incl. `itemHome`, `init`, `live`, `broadcast`, `overlay.closeAria`, `attendance`) | §5.3, §5.4, §5.5 |
| `src/app/ui/components/meeting/hooks/useMeetingPage.tsx` | `MeetingPage` type + `MeetingPageProvider` + `useMeetingPage` (`useState("init")`) | §5.5 |
| `src/app/ui/components/meeting/hooks/useMeetingLive.tsx` | `useMeetingLiveInstance` + `MeetingLiveProvider` + public `useMeetingLive`; SyncedStore root `{ [MEETING_LIVE_MAP]: {} }` as `Partial<MeetingLiveMap>`; `/meeting` session; `connect_error` linking branch | §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveMe.ts` | current-member `participants` proxy (no clone); used by session for `me` / `can` / `actions` | §5.2 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | session surface: resolve + provider; one `useMeetingAttendWindow` call; exposes `attendWindow` | §5.3, §5.5 |
| `src/app/ui/components/meeting/hooks/useMeetingAttendWindow.ts` | attend-window clock (mirror math private; open timeout + 30s ticks while waiting) | §5.3, §5.5 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts` | LiveKit join fetch — union `{ status, token, url }`; quiet network retry; only caller is the room hook | `../backend/contracts/livekit-media-plane.md` §6.5; `flow-meeting-broadcast.md` §4 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitRoom.ts` | `Room.connect` owner: status projection, peer map, publish toggles, sound autoplay gate, cooperative mute-all, lifecycle | `flow-meeting-broadcast.md` §5 |
| `src/resources/configs/axios/api.ts` | `CUSTOM.ORG_LIVEKIT_TOKEN` → `/custom/org/livekit_token` | `../backend/contracts/livekit-media-plane.md` §6.1 |
| `src/resources/configs/axios.ts` | honor `skipNetworkToast` on network reject | `../backend/contracts/livekit-media-plane.md` §6.5 |
| `src/types/extends/global.ts` | `AxiosRequestConfig.skipNetworkToast` | `../backend/contracts/livekit-media-plane.md` §6.5 |
| `src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | PENDING / FAILED gate UI via session `linking` (`Loadable` / shared `LaneFailed`) | §5.3 |
| `src/app/ui/components/Wrong.tsx` | `LaneFailed` optional `title` / `description` (fallback to `wrong.laneFailed`) | §5.3; `flow-meeting-broadcast.md` §8 |
| `src/types/meeting.ts` | mirrored live map (`MeetingLiveMap`, `participants`, `MEETING_LIVE_*`; pair with `backend/src/app/types/meeting.ts`) | §5.1; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `package.json` | `yjs` + `@syncedstore/core` + `@syncedstore/react` + `livekit-client` exact pins | §5.1; `flow-meeting-broadcast.md` §3 |
| `src/app/ui/layouts/MeetingLayout.tsx` | `MEETING` layout — live → session → page providers, linking gate, branded shell, READY `FlexContainer` content column | §5, §5.1, §5.3, §5.4, §5.5 |
| `src/app/ui/base/core/MyApp.tsx` | `case "MEETING"` → `MeetingLayout` | §5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | org branding hook; `OrganizationColors` (incl. `idleCardBackground` / `presentCardBackground`, fixed-white `actionIconOnFill`); light `softLight` mix 0.78 for section chips | §5.4 |
| `src/app/ui/components/meeting/MeetingHeader.tsx` | menu + org logo/name; `MeetingHeaderMe` when `me` exists; disabled request-to-speak (`requestTalk` / `requestTalkAria`) for MEMBER and VIEWER; accent rail; `fixed` mobile bar | §5.4 |
| `src/app/ui/components/meeting/MeetingHeaderMe.tsx` | current participant avatar + name + type chip (org colors, session `me`) | §5.4 |
| `src/app/ui/components/meeting/MeetingFooter.tsx` | platform rights line (no unjustified top margin) | §5.4 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | Meeting info after `live`; role tile grid (`wide` live); live corner ping while `STARTED`; disable locked `live` (`ariaLabel`); `setPage` + selected accent chrome; prefs row | §5.4, §5.5 |
| `src/app/ui/components/meeting/MeetingDrawerOverlay.tsx` | mobile portal overlay | §5.4 |
| `src/app/ui/components/HeaderIconButton.tsx` | top-level shared icon button; consumed by `CustomerHeader` + `MeetingHeader` | §5.4; `component-structure.md` §3 |
| `src/app/ui/components/customer/CustomerHeader.tsx` | consumes `HeaderIconButton` from the shared top level | §5.4 |
| `src/app/ui/components/DrawerMenuIcon.tsx` | optional `accentClr` prop; default is the accent bar | §5.4 |
| `src/app/ui/components/HomeMark.tsx` | shared home mark; optional `frameClr` / `accentClr` (meeting Home forces both to `actionIconOnFill`) | §5.4; `flow-customer-shell.md` |
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
| `src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts` | LiveKit join-token (`org_host` + peek `STARTED` + reuse-or-mint); returns `{ token, url }` | `../backend/contracts/livekit-media-plane.md` §6 |
| `src/app/http/routes/website.ts` | `OrgCustomRouter` + `POST /custom/org/start` + `POST /custom/org/livekit_token` | §6.1; `../backend/contracts/livekit-media-plane.md` §6.1 |
| `src/app/orm/models/MeetingParticipant.ts` | roster join + optional `livekit_token` / `livekit_token_expires_at` | `../backend/contracts/meeting-participant-domain.md`; `livekit-media-plane.md` §6 |
| `src/resources/trans/ar/messages.ts`, `src/resources/trans/en/messages.ts` | `MEETING_NOT_LIVE` | `../backend/contracts/livekit-media-plane.md` §7.2 |
| `src/app/http/middlewares/OrganizationHostMiddleware.ts` | header-based org resolve + `currentOrganization` | §6.2, §8 |
| `src/resources/configs/http/middlewares/index.ts` | `org_host` registration | §6.2 |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | actor handshake `token` + `SocketData` for `/customer`, `/supervisor` | `../backend/contracts/meeting-realtime-socket.md` §2 |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | `/meeting` handshake + `MeetingSocketData` + `current*` helpers | `../backend/contracts/meeting-realtime-socket.md` §2 |
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | shared socket typing, bound-event set, `rejectLive` | `../backend/contracts/meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | join `Rooms.MEETING`; bind the live event set | `../backend/contracts/meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | sync handshake + server state vector | `../backend/contracts/meeting-realtime-socket.md` §3.1 |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | validation, status gate, apply, room broadcast | `../backend/contracts/meeting-realtime-socket.md` §3.2 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | private codec+seed (incl. nested `participants` + `datetime`) + registry + BLOB persistence | `../backend/contracts/meeting-live-state.md` §1.3, §2 |
| `src/app/types/meeting.ts` | mirrored `MeetingLiveMap` / `MEETING_LIVE_*` / participant unions (pair with `website/src/types/meeting.ts`; includes `datetime`) | §5.1; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `src/app/orm/models/Meeting.ts` | `live_state` column; schedule + `ATTEND_OPEN_BEFORE_MS` statics | `../backend/contracts/meeting-live-state.md` §1.1; `meeting-domain.md` §3.2b |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | `live_state` excluded from GQL attrs | `../backend/contracts/meeting-live-state.md` §4 |
| `src/resources/configs/socket/io.ts` | `/meeting` namespace + `meeting_auth` + meeting controllers | `../backend/contracts/meeting-realtime-socket.md` §1 |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM / room | `../backend/contracts/meeting-realtime-socket.md` §4 |
| `.env.example` | `TEST_ORGANIZATION_ID=1` | §9 |

### Governance

| Path | Role |
|---|---|
| `.cursor/rules/organization-host-routing.mdc` | host mode, route gate, transport identification |
| `.cursor/rules/livekit-media-plane.mdc` | LiveKit helper + join-token HTTP + participant JWT cache |
| `.cursor/rules/meeting-participant-roster.mdc` | roster join + LiveKit token columns never on GQL |
| `.cursor/rules/meeting-realtime-socket.mdc` | Meeting session placement, hook contract, `connect_error`, backend pairing |
| `.cursor/rules/website-meeting-live-session.mdc` | `MeetingLiveSessionProvider` / `useMeetingLiveSession` linking/can/actions/meeting/me + linking gate UI |
| `.cursor/rules/website-meeting-shell.mdc` | READY shell drawer IA + in-shell `MeetingPage` + header request-to-speak |
| `.cursor/rules/meeting-live-state.mdc` | CRDT document ownership, V2 codec, BLOB exposure |
| `.cursor/rules/meeting-live-map-mirror.mdc` | identical `MeetingLiveMap` files on backend ↔ website |
| `.cursor/rules/sequelize-include-by-association-name.mdc` | Sequelize `include` by association name (backend companion) |
| `.cursor/rules/nodejs-socket-namespace-registration.mdc` | namespace / controller / room registration |
| `.cursor/rules/nodejs-socket-handler-contract.mdc` | connection return = absolute listener set |
| `.cursor/rules/socket-event-mirroring.mdc` | outbound mirror scope |
| `.cursor/rules/website-semantic-color-token-discipline.mdc` | runtime per-tenant color maps — shell keys reference `semanticColor`, only brand keys are computed (§5.4) |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | Meeting realtime transport workflow |
| `.cursor/skills/meeting-livekit-token/SKILL.md` | LiveKit join-token HTTP + website fetch hook |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | Meeting session surface + linking gate workflow |
| `.cursor/skills/website-meeting-shell/SKILL.md` | Meeting READY shell + in-shell `MeetingPage` workflow |
| `.cursor/skills/nodejs-socket-server-event/SKILL.md` | backend socket surface workflow |

## 10a) Change set inventory (flat MeetingLiveSession + nested session provider)

Historical delivery for the product session surface: drop nested `stages` remaps; ship `{ linking, can, actions, meeting, me }` via `MeetingLiveSessionProvider` under `MeetingLiveProvider`. Path map for the whole org-host contract remains in §10. In-shell `MeetingPage` is §10b / §5.5.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/meetingLiveSession.ts` | modified — `MeetingLiveLinking`; state = `{ linking, can }`; no meeting/me stage remaps | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.ts` | deleted — replaced by `.tsx` provider module | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | added (rename/replace) — `useMeetingLiveSessionInstance` + `MeetingLiveSessionProvider` + context `useMeetingLiveSession` | §5.3 |
| `src/app/ui/layouts/MeetingLayout.tsx` | modified — nest `MeetingLiveSessionProvider`; gate on `linking` | §5.3, §5.4 |
| `src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | modified — reads `linking` from session hook (no prop) | §5.3 |
| `src/app/ui/components/meeting/MeetingHeader.tsx` | modified — `me` from `useMeetingLiveSession()` | §5.4 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | modified — `me` from `useMeetingLiveSession()` | §5.4 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.1–§5.4 session surface + write contract + inventory | — |
| `docs/platforms/backend/contracts/meeting-realtime-socket.md` | product linking via `useMeetingLiveSession().linking` | §3.3 there |
| `docs/platforms/website/component-structure.md` | live-meeting row names session provider surface | component index |
| `docs/platforms/website/README.md` | overview / governance pointers for session provider | change-set table |
| `.cursor/rules/website-meeting-live-session.mdc` | provider nesting, flat surface, write contract | governance |
| `.cursor/rules/website-meeting-shell.mdc` | READY on `linking`; role from session `me` | governance |
| `.cursor/rules/meeting-realtime-socket.mdc` | session reader under nested provider | governance |
| `.cursor/rules/organization-host-routing.mdc` | public UI names session provider | governance |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | session provider workflow | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | session `me` for role IA | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | defers product session to live-session skill | governance |

## 10b) Change set inventory (in-shell `MeetingPage` + init lobby)

Current delivery: local `MeetingPage` state (`"init"` default), `MeetingPageProvider` in the layout chain, drawer `setPage`, page switch in `Meeting.tsx`, shipped `MeetingInitPage` lobby (org identity, meta chips, attend/attended/left), light-scheme section tint mix **0.78**, drawer `itemLive` copy = Meeting room / غرفة الاجتماع. Full path map remains in §10; behavior in §5.4–§5.5.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useMeetingPage.tsx` | added — `MeetingPage` + `MeetingPageProvider` + `useMeetingPage` | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — lobby UI (org identity, meta chips, attend / attended / left) | §5.5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | modified — `softLight` default mix **0.78** (was 0.92) for light section tints | §5.4 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `meetingLayout.init` keys; `drawer.itemLive` → غرفة الاجتماع / Meeting room | §5.4, §5.5 |
| `src/app/ui/pages/Meeting.tsx` | modified — `MeetingContent` switch on `page` | §5, §5.5 |
| `src/app/ui/layouts/MeetingLayout.tsx` | modified — nest `MeetingPageProvider` under session provider | §5.5 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | modified — local `MeetingDrawerPage`; tiles `setPage` + selected chrome; removed `available={false}` | §5.4, §5.5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.4 softLight + brand consumers; §5.5 lobby; §8; §10b | — |
| `docs/platforms/website/component-structure.md` | meeting shell / page index rows | component index |
| `docs/platforms/website/README.md` | known limits + change-set pointers | overview |
| `.cursor/rules/website-meeting-shell.mdc` | in-shell `MeetingPage`, `MeetingInitPage` lobby, drawer `live` label | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | page provider + init lobby + `meeting/pages` workflow | governance |
| `.cursor/rules/organization-host-routing.mdc` | public UI names `MeetingPageProvider` | governance |

## 10c) Change set inventory (Home control + drawer stubs + selected chrome)

Current delivery on top of §10b: full-row **Home** → `"init"` (`itemHome` + monochrome `HomeMark`), named drawer page stubs wired in `Meeting.tsx`, fixed-white `actionIconOnFill` for solid wells, selected tile chrome = soft `sectionAccentBackground` + partial start rail + `textAccent` (icon well unchanged). Full path map remains in §10; behavior in §5.4–§5.5.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/HomeMark.tsx` | modified — optional `frameClr` / `accentClr` (default accent floor unchanged for customer/landing) | §5.4; `flow-customer-shell.md` |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | modified — Home control (`mark` branch); selected chrome (soft accent + partial rail); always-white well glyphs via `actionIconOnFill` | §5.4, §5.5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | modified — brand key `actionIconOnFill` = `#FFFFFF` both schemes | §5.4 |
| `src/app/ui/components/meeting/pages/MeetingPageStub.tsx` | added — shared title-only placeholder | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingLivePage.tsx` | added — `"live"` stub | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingTalkQueuePage.tsx` | added — `"talkQueue"` stub | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingAttendancePage.tsx` | added — `"attendance"` stub | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingAgendaPage.tsx` | added — `"agenda"` stub | §5.5, §8 |
| `src/app/ui/components/meeting/pages/MeetingDecisionsAndVotePage.tsx` | added — `"decisionsAndVote"` stub | §5.5, §8 |
| `src/app/ui/pages/Meeting.tsx` | modified — switch mounts named stub pages (no longer `null` for drawer ids) | §5.5 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `drawer.itemHome` = الرئيسية / Home | §5.4 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.4 selected chrome + Home; §5.5 stubs; §8; §10c | — |
| `docs/platforms/website/flow-customer-shell.md` | `HomeMark` optional color overrides | shared mark API |
| `docs/platforms/website/shared-ui-and-shell.md` | `HomeMark` / `DrawerMenuIcon` override props | shared marks |
| `docs/platforms/website/component-structure.md` | meeting `pages/*` stub index | component index |
| `docs/platforms/website/README.md` | known limits / shell pointers | overview |
| `.cursor/rules/website-meeting-shell.mdc` | Home + selected chrome + stubs | governance |
| `.cursor/rules/website-customer-utils-composed-marks.mdc` | `HomeMark` `frameClr`/`accentClr`; meeting monochrome-on-fill | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | Home + selected chrome + stub replacement workflow | governance |

## 10d) Change set inventory (attend window + Meeting room gate)

Current delivery on top of §10a–§10c: live map `datetime` seed; `MeetingModel.ATTEND_OPEN_BEFORE_MS`; website attend-window hook + `can.attend` open window + `can.enterLive`; init remaining-duration copy only while the window is not yet open; disabled drawer `live` tile (`ariaLabel`); `MeetingLivePage` bounce. **Superseded ownership details:** §10g (single session clock + `attendWindow`). Full path map remains in §10; behavior in §5.3–§5.5 and backend live-state / participant / meeting-domain contracts.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — `MeetingLiveMap.datetime` | §5.1; `meeting-live-state.md` §1.2 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `datetime` on create (no BLOB backfill) | `meeting-live-state.md` §1.3 |
| `src/app/orm/models/Meeting.ts` | modified — `ATTEND_OPEN_BEFORE_MS` | `meeting-domain.md` §3.2b |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — mirror `datetime` | §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingAttendWindow.ts` | added — attend-window clock (later owned solely by session; see §10g) | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | modified — `datetime` / open window via attend hook; see §10g for `attendWindow` | §5.3 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — remaining attend copy; room hints (final: reads session `attendWindow`, §10g) | §5.5 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | modified — disable `live` when `!can.enterLive`; locked `ariaLabel` = `init.attendRequiresForRoom` | §5.4 |
| `src/app/ui/components/meeting/pages/MeetingLivePage.tsx` | modified — bounce to `"init"` when locked | §5.5 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `attendAvailableIn`, `attendRequiresForRoom`, `roomUnlockedHint` | §5.5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.3–§5.5, §8, §10, §10d | — |
| `docs/platforms/backend/contracts/meeting-live-state.md` | `datetime` on live map + seed | live-state contract |
| `docs/platforms/backend/contracts/meeting-participant-domain.md` | self-check-in open window | §3.6 there |
| `docs/platforms/backend/contracts/meeting-domain.md` | `ATTEND_OPEN_BEFORE_MS` | §3.2b there |
| `.cursor/rules/website-meeting-live-session.mdc` | attend window + `enterLive` + clock refresh | governance |
| `.cursor/rules/website-meeting-shell.mdc` | init remaining only pre-window; disabled `live` + `ariaLabel` | governance |
| `.cursor/rules/website-backend-policy-mirror.mdc` | attend-window hook owned by session (`attendWindow`) | governance |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | session attend / enterLive workflow | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | init + room gate workflow | governance |

## 10e) Change set inventory (MeetingHeader identity)

Current delivery on top of §10a–§10d: READY-shell header shows the current live participant via presentational `MeetingHeaderMe` (session `me` fields + org colors). No backend / live-type / GQL change. Full path map remains in §10; behavior in §5.4.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/MeetingHeaderMe.tsx` | added — avatar + name + type chip | §5.4 |
| `src/app/ui/components/meeting/MeetingHeader.tsx` | modified — mount `MeetingHeaderMe` when `me`; trailing `ml` ownership with request-talk | §5.4 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `header.typeChairperson` / `typeMember` / `typeViewer` | §5.4 i18n table |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.4 header identity, §8/§10 tables, §10e | — |
| `docs/platforms/website/component-structure.md` | meeting-shell row lists `MeetingHeaderMe` | component inventory |
| `.cursor/rules/website-meeting-shell.mdc` | Header identity + request-to-speak; globs include `MeetingHeaderMe.tsx` | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | header identity step | governance |

## 10f) Change set inventory (chair attendance log)

Current delivery on top of §10a–§10e: live map `minMembersCount` seed (new empty `live_state` only); chair-only `MeetingAttendancePage` (quorum strip, filters, responsive grid tiles); init attended copy = first-person relative duration + quieter room hint. Full path map remains in §10; behavior in §5.5 and `meeting-live-state.md` §1.2 / §9c.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — `MeetingLiveMap.minMembersCount` | §5.1; `meeting-live-state.md` §1.2 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `minMembersCount` on create (no BLOB backfill) | `meeting-live-state.md` §1.3 |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — identical mirror | §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingAttendance.ts` | added — attendance data hook (buckets/sort/quorum + view-model) | §5.5 |
| `src/app/ui/components/FilterCountChips.tsx` | added — shared counted filter chips (presentational) | §5.5 |
| `src/app/ui/components/meeting/MeetingAttendanceCard.tsx` | added — attendance participant card (own chrome; not HeaderMe) | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingAttendancePage.tsx` | modified — chair attendance log (was stub) | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — first-person relative `attendedAt`; `roomUnlockedHint` → `textTertiary` | §5.5 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `meetingLayout.attendance.*`; init `attendedAt` first person | §5.5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.5 attendance, §8, §10f | — |
| `docs/platforms/backend/contracts/meeting-live-state.md` | `minMembersCount` on live map + seed | live-state §1.2, §9c |
| `docs/platforms/website/component-structure.md` | meeting pages note attendance product UI | component inventory |
| `.cursor/rules/website-meeting-shell.mdc` | attendance page + init copy voice | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | attendance log workflow | governance |

## 10g) Change set inventory (hook ownership cleanup + single attend clock)

Final delivery on top of §10a–§10f after structure review: delete loose `meetingAttendWindow.ts` / `meetingLiveSession.ts` (logic lives in hook modules); attend-window mirror is **hook-private** and called **once** from session; session public shape adds `attendWindow`; Init reads that field only; chair bounce lives on `MeetingAttendancePage` (not in `useMeetingAttendance`); shared `FilterCountChips`; attendance pure helpers colocated in `useMeetingAttendance.ts`. Behavior authority remains §5.3–§5.5 and backend live-state / meeting-domain contracts.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — `MeetingLiveMap.minMembersCount` | §5.1; `meeting-live-state.md` §1.2, §9c |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `minMembersCount` on first empty create only (no BLOB backfill) | `meeting-live-state.md` §1.3, §9c |
| `src/app/orm/models/Meeting.ts` | already has `ATTEND_OPEN_BEFORE_MS` (prior ship; not dirty in this inventory) | `meeting-domain.md` §3.2b |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — identical `minMembersCount` mirror | §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingAttendWindow.ts` | **added** — single attend-window clock; mirror math private | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | modified — resolve takes `windowOpen`; instance owns one clock; exposes `attendWindow`; deleted reliance on loose `meetingLiveSession.ts` | §5.3 |
| `src/app/ui/components/meeting/meetingAttendWindow.ts` | **deleted** — replaced by hook module | §5.3 |
| `src/app/ui/components/meeting/meetingLiveSession.ts` | **deleted** — resolve/colocation moved into `useMeetingLiveSession.tsx` | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingAttendance.ts` | **added** — buckets/sort/quorum + filter/rows view-model; **no** navigation | §5.5 |
| `src/app/ui/components/FilterCountChips.tsx` | **added** — shared presentational counted chips (no meeting i18n) | §5.5; `component-structure.md` §3 |
| `src/app/ui/components/meeting/MeetingAttendanceCard.tsx` | **added** — roster card chrome (not `MeetingHeaderMe`); fills later use org tokens (§10h) | §5.5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | modified in §10h — `idleCardBackground` / `presentCardBackground` | §5.4, §10h |
| `src/app/ui/components/meeting/pages/MeetingAttendancePage.tsx` | modified — chair product UI; page owns non-chair bounce | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — reads `attendWindow` from session; present via `can.enterLive`; first-person relative attended copy | §5.5 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `meetingLayout.attendance.*`; init first-person `attendedAt` / `attendedTitle` | §5.5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.3 single clock + `attendWindow`, §5.5 attendance, §10g | — |
| `docs/platforms/website/component-structure.md` | FilterCountChips + meeting hook rows | component inventory |
| `docs/platforms/backend/contracts/meeting-live-state.md` | `minMembersCount` seed contract | live-state §1.2, §9c |
| `docs/platforms/backend/contracts/meeting-domain.md` | `ATTEND_OPEN_BEFORE_MS` → website hook + session | §3.2b |
| `.cursor/rules/website-meeting-live-session.mdc` | single clock; `attendWindow` on session; no page re-call | governance |
| `.cursor/rules/website-meeting-shell.mdc` | attendance bounce on page; FilterCountChips; card ≠ HeaderMe | governance |
| `.cursor/rules/website-backend-policy-mirror.mdc` | attend-window hook ownership | governance |
| `.cursor/rules/meeting-realtime-socket.mdc` | product math belongs in `useMeetingLiveSession` (path fix) | governance |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | attendWindow workflow | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | attendance page + bounce + chips | governance |

## 10h) Change set inventory (attendance card org color tokens)

On top of §10g: attendance idle/present fills are dedicated `OrganizationColors` scheme pairs — no component-level `colorScheme` patches. Present card must not reuse `sectionAccentBackground` (type chip fill).

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | modified — `idleCardBackground` (`#FFFFFF @transparent`); `presentCardBackground` (softLight **0.62** / softDark **0.18**) | §5.4 |
| `src/app/ui/components/meeting/MeetingAttendanceCard.tsx` | modified — present → `presentCardBackground`; idle → `idleCardBackground`; type chip stays `sectionAccentBackground` | §5.5 |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.4 token split + §5.5 card fills + §10h | — |
| `.cursor/rules/website-meeting-shell.mdc` | create-token-not-patch; attendance card fills | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | attendance color tokens workflow | governance |

## 10i) Change set inventory (Init attend CTA caption)

On top of §10h: when `can.attend`, Init shows primary attend button **plus** `attendRequiresForRoom` caption under it (same key as the waiting strip and locked `live` tile aria). No new translation keys.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — `can.attend` branch: CTA + `attendRequiresForRoom` caption | §5.5 |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.5 attendance branch (1); §10i | — |
| `.cursor/rules/website-meeting-shell.mdc` | Init attend CTA + caption | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | Init attend CTA + caption step | governance |

## 10j) Change set inventory (Meeting room waiting / start / broadcast shell)

On top of §10i: replace `MeetingLivePage` title stub with status-gated product body; empty/placeholder `MeetingLiveBroadcast`; shared `MeetingPrimaryButton` for Init attend + Live start; `meetingLayout.live` copy; align website `meetingStatus` / `meetingType` display strings with backend enum labels (`STARTED` = بدأ / Started).

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/pages/MeetingLivePage.tsx` | modified — enterLive bounce; `STARTED` → broadcast; else waiting + `can.startMeeting` CTA | §5.5 |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | **added** — placeholder broadcast shell (`broadcastPlaceholder`; both superseded by the real stage in §10m, key deleted) | §5.5 |
| `src/app/ui/components/meeting/MeetingPrimaryButton.tsx` | **added** — org primary full-width CTA | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — attend CTA uses `MeetingPrimaryButton` | §5.5 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `meetingLayout.live.*`; enum label alignment for `statusStarted` / related STARTED copy | §5.5 |
| `lib/tsconfig.tsbuildinfo` | modified — TypeScript incremental build info (generated; not behavioral) | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.5 live body; §8; §10 / §10j | — |
| `.cursor/rules/website-meeting-shell.mdc` | Meeting room body + `MeetingPrimaryButton` | governance |
| `.cursor/rules/website-backend-enum-label-mirror.mdc` | website enum display labels must match backend `general.ts` enums | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | Meeting room + primary button steps | governance |

## 10k) Change set inventory (Init structure + MeetingMetaChip)

On top of §10j: extract shared org `MeetingMetaChip`; replace Init nested-ternary / `renderAttendSection` with colocated page-only `InitAttendSection` component; keep page-only sections in the page file and shared chrome under `meeting/`.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/MeetingMetaChip.tsx` | **added** — shared org type/status chip (`label` + `tone`) | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — `InitAttendSection` colocated; meta uses `MeetingMetaChip` | §5.5 |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.5 Init structure; §10k | — |
| `.cursor/rules/website-meeting-shell.mdc` | `MeetingMetaChip` + page-local section placement | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | InitAttendSection / MetaChip steps | governance |

## 10l) Change set inventory (persistent broadcast + solid page overlay + Meeting info tile)

On top of §10k: while `STARTED`, `Meeting.tsx` keeps `MeetingLiveBroadcast` mounted and floats non-`live` pages in solid `MeetingPageOverlay` (floating close; dismiss → `"live"`); `MeetingLivePage` is waiting/start only; drawer Meeting info (`itemHome`) sits full-row after `live`; `live` tile corner ping while `STARTED` + enterLive; READY content column is `FlexContainer` (align with header/footer `Container`); exclusive page bodies use `pv` only; overlay scroll body adds `ph={page.padY}`; `meetingLayout.overlay.closeAria`.

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/pages/Meeting.tsx` | modified — `STARTED` stack: persistent broadcast + overlay for non-`live`; bounce when `STARTED` + `live` + `!enterLive` | §5.5 |
| `src/app/ui/components/meeting/MeetingPageOverlay.tsx` | **added** — solid sheet full content-column width; vertical float inset; scroll `ph=page.padY`; floating close; outside tap dismiss | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingLivePage.tsx` | modified — waiting/start only (no broadcast branch); `pv` only | §5.5 |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | modified — mount owner is `Meeting.tsx` while `STARTED`; `pv` only | §5.5 |
| `src/app/ui/components/meeting/pages/MeetingInitPage.tsx` | modified — `pv` only (no horizontal pad; overlay supplies `ph` when floating) | §5.4, §5.5 |
| `src/app/ui/components/meeting/pages/MeetingAttendancePage.tsx` | modified — drop `ph={page.padX}`; `pv` only | §5.4, §5.5 |
| `src/app/ui/components/meeting/pages/MeetingPageStub.tsx` | modified — `pv` only | §5.5 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | modified — Meeting info after `live`; live corner ping; always-filled tile `buttonReset` | §5.4 |
| `src/app/ui/layouts/MeetingLayout.tsx` | modified — READY children in `FlexContainer`; flex/`minH={0}` for Absolute overlay height | §5.4 |
| `src/app/ui/components/meeting/MeetingFooter.tsx` | modified — drop unjustified `mt={3}` | §5.4 |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `drawer.itemHome` Meeting info / معلومات الاجتماع; `overlay.closeAria` | §5.3–§5.5 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/organization-host-routing.md` | this page — §5.4–§5.5 overlay stack + column width; §8; §10 / §10l; §11 | — |
| `docs/platforms/website/component-structure.md` | meeting shell index includes broadcast + overlay + primary/meta; `Meeting` page STARTED stack note | component index |
| `docs/platforms/website/README.md` | organization-host change-set pointer → §10l | platform index |
| `.cursor/rules/website-meeting-shell.mdc` | STARTED broadcast stack + solid overlay + FlexContainer column + overlay `ph` + Meeting info after live | governance |
| `.cursor/skills/website-meeting-shell/SKILL.md` | same IA / overlay / column pad steps | governance |

## 10m) Change set inventory (LiveKit broadcast A/V)

On top of §10l: backend join-token returns `{ token, url }`; `useMeetingLiveKitToken` becomes a discriminated union; new `useMeetingLiveKitRoom` owns `Room.connect` and all media state; `MeetingLiveBroadcast` replaces the token probe with the real stage (featured chair + remote grid, camera/mic/**sound** controls, chair mute-all); session gains gate-only `can.muteAllMedia`; `LaneFailed` accepts `title` / `description` and now serves both broadcast failure and linking FAILED; `meetingLayout.broadcast.*` + `linking.failedHint` copy.

Full behavior contract: `flow-meeting-broadcast.md`.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts` | modified — `Result` + step 7 return `url` from `LiveKitHelper.clientUrl()` | §10 backend table; `../backend/contracts/livekit-media-plane.md` §6.2–§6.3 |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitRoom.ts` | **added** — room lifecycle, status projection, peers, publish/playback, mute-all data channel | `flow-meeting-broadcast.md` §5 |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | modified — real A/V stage (attach components, tiles, controls, chair group); probe removed | §5.5; `flow-meeting-broadcast.md` §6 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts` | modified — `{ status, token, url }` union | `flow-meeting-broadcast.md` §4 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | modified — `muteAllMedia` capability (type + `canNone` + `resolveCan`) | §5.3; `flow-meeting-broadcast.md` §7 |
| `src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | modified — FAILED early return renders `LaneFailed`; PENDING unchanged | §5.3; `flow-meeting-broadcast.md` §8 |
| `src/app/ui/components/Wrong.tsx` | modified — `LaneFailed` optional `title` / `description` | `flow-meeting-broadcast.md` §8; `shared-ui-and-shell.md` |
| `src/resources/translations/ar.ts`, `en.ts` | modified — `meetingLayout.broadcast.*` (16 keys) + `linking.failedHint` added; `failedMessage` punctuation; orphan `live.broadcastPlaceholder` removed | §5.3–§5.5; `flow-meeting-broadcast.md` §9 |
| `package.json`, `yarn.lock` | modified — `livekit-client@2.21.0` exact + transitive entries | `flow-meeting-broadcast.md` §3 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/flow-meeting-broadcast.md` | **added** — broadcast client contract | — |
| `docs/platforms/website/organization-host-routing.md` | this page — §5.3 capability + FAILED chrome, §5.5 broadcast body, §8 limits 11–12, §10 / §10m, §11 | — |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | `{ token, url }`, hook union, probe removed, client pointer | backend contract |
| `docs/platforms/backend/contracts/client-portal-http-website.md` | response shape + hook chain | HTTP index |
| `docs/platforms/backend/modules/runtime-integrations.md` | §7b response + client chain | integration index |
| `docs/platforms/backend/README.md` | LiveKit row mentions the browser client | backend index |
| `docs/platforms/website/README.md` | flow index + change-set pointer → §10m | website index |
| `docs/platforms/website/component-structure.md` | meeting shell list: room hook + broadcast stage | component index |
| `docs/platforms/website/shared-ui-and-shell.md` | `LaneFailed` override props | shared UI |
| `docs/invariants/website.md` | **W60** media element ownership | invariants |
| `docs/README.md` | website doc index row | root index |
| `.cursor/rules/website-meeting-livekit-broadcast.mdc` | **added** — browser-client invariants | governance |
| `.cursor/rules/website-meeting-shell.mdc` | pointer to the broadcast rule | governance |
| `.cursor/rules/livekit-media-plane.mdc` | hook API + response, probe removed | governance |
| `.cursor/rules/website-meeting-live-session.mdc` | FAILED chrome + `muteAllMedia` gate-only | governance |
| `.cursor/skills/website-meeting-broadcast/SKILL.md` | **added** — broadcast workflow | governance |
| `.cursor/skills/meeting-livekit-token/SKILL.md` | `{ token, url }`, probe removed, hand-off | governance |

## 10n) Change set inventory (live agenda map fields)

On top of §10m: mirrored `MeetingLiveMap` gains nested `agendaItems` + root `currentAgendaItemId`. Backend seeds from SQL on first empty `live_state` only (`status: "WAITING"`, `isLiveCreated` / `isLiveUpdated: false`, `currentAgendaItemId: null`). In-session cancel is `status: "CANCELED"` (no live delete). Writers and agenda page UI are **not** shipped. Authority: `../backend/contracts/meeting-live-state.md` §1.2 / §1.3 / §9d; `../backend/contracts/agenda-item-domain.md`.

### Website

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — identical live map mirror (`MeetingLiveAgendaItem*`, `agendaItems`, `currentAgendaItemId`) | §5.1; `meeting-live-state.md` §1.2, §9d |
| `lib/tsconfig.tsbuildinfo` | modified — incremental TS cache from type-check | **excluded** (generated) |

### Backend (sibling repo)

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — live agenda types + map fields | `meeting-live-state.md` §1.2, §9d |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — `buildLiveAgendaItems`; nested seed; `currentAgendaItemId` after `agendaItems` | `meeting-live-state.md` §1.3, §9d |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-live-state.md` | modified — agenda shape, seed, §9d inventory + triage | backend contract |
| `docs/platforms/backend/contracts/agenda-item-domain.md` | modified — SQL vs live session fields | backend contract |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | modified — plane table notes live map vs LiveKit for agenda | backend contract |
| `docs/platforms/website/organization-host-routing.md` | this page — §5.1 / shipped limits / §10n | — |
| `docs/platforms/website/README.md` | change-set pointer → §10n | website index |
| `.cursor/rules/meeting-live-state.mdc` | modified — nested agenda; map-only session fields | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | modified — mirror includes agenda | governance |
| `.cursor/rules/agenda-item-meeting-child.mdc` | modified — live session fields | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | modified — agenda seed/mirror checklist | skill |

## 10o) Change set inventory (live talk queue fields)

On top of §10n: per-participant `talkTurn` (`null` = not queued) and root `currentTalkMemberId` (`null` = nobody speaking; set after `participants` in `createLiveDoc`). Session-only; durable talk history stays SQL `TalkRecord`. Writers / talk-queue UI / header request-to-speak are **not** shipped. Authority: `../backend/contracts/meeting-live-state.md` §1.2 / §9e; `../backend/contracts/talk-record-domain.md`.

### Website

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — `talkTurn` on `MeetingLiveParticipant`; `currentTalkMemberId` on `MeetingLiveMap` | §5.1; `meeting-live-state.md` §1.2, §9e |
| `lib/tsconfig.tsbuildinfo` | may change from type-check | **excluded** (generated) |

### Backend (sibling repo)

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — identical mirror | `meeting-live-state.md` §1.2, §9e |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `talkTurn: null`; `currentTalkMemberId` after `participants` | `meeting-live-state.md` §1.3, §9e |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-live-state.md` | modified — talk fields + §9e inventory | backend contract |
| `docs/platforms/backend/contracts/talk-record-domain.md` | modified — live vs SQL talk fields | backend contract |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | modified — plane table notes live talk fields | backend contract |
| `docs/platforms/website/organization-host-routing.md` | this page — §5.1 / §8 / §10o | — |
| `docs/platforms/website/README.md` | change-set pointer → §10o | website index |
| `docs/platforms/backend/README.md` | live-state / talk-record index blurbs | backend index |
| `docs/README.md` | live-state index blurb | root index |
| `.cursor/rules/meeting-live-state.mdc` | modified — talk session fields + seed order | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | modified — mirror includes talk fields | governance |
| `.cursor/rules/talk-record-meeting-child.mdc` | modified — live vs SQL | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | modified — talk seed checklist (6d) | skill |

## 11) Verification

- `yarn type-check` in `website/` and in `backend/`.
- Diff `backend/src/app/types/meeting.ts` against `website/src/types/meeting.ts` (must stay identical).
- Apex host: `/` boots through `API.CUSTOM.START`; `/meeting/...` renders `Error` `404`.
- Organization host: boot calls `org/start` and hydrates `organizationHost`; `/meeting/...` mounts `MEETING` layout (`MeetingLiveProvider` → `MeetingLiveSessionProvider` → `MeetingPageProvider`) + linking gate; after READY, branded shell with header identity (`MeetingHeaderMe` from session `me`) + `MeetingInitPage` lobby (attend CTA + `attendRequiresForRoom` caption when `can.attend`; remaining-duration copy before the open window; Meeting room requires check-in); READY content column is `FlexContainer` (aligns with header/footer `Container`); drawer Meeting info returns to `"init"`; drawer `live` disabled until `can.enterLive` (corner ping while `STARTED`); Meeting room shows waiting + chair start when not started; while `STARTED`, persistent `MeetingLiveBroadcast` with other pages in solid `MeetingPageOverlay` (`ph=page.padY` matches page `pv`); chair `attendance` mounts the attendance log (non-chair bounce to `"init"`); other drawer ids mount title stubs; `/customer/...` renders `Error` `404`.
- Init lobby light chips: `sectionBrandBackground` / `sectionAccentBackground` readable on `pageBackground` (org `softLight` mix 0.78).
- Selected drawer tile: soft `sectionAccentBackground` + partial start accent rail + `textAccent`; icon well stays primary (white glyph / white `HomeMark`).
- Attendance cards: present fill uses `presentCardBackground` (distinct from type chip `sectionAccentBackground`); idle fill uses `idleCardBackground` (light card / dark transparent).
- Socket: organization host opens **no** boot socket; `MeetingLayout` opens `/meeting` once via `MeetingLiveProvider` / `useMeetingLiveInstance`, joins `meeting-{id}`, and emits `meeting.live.sync`; apex authed customer still connects to `/customer`.
- Broadcast (`STARTED`, two browsers): each side sees the other's tile; camera off → avatar tile; sound starts muted and needs one click; chair mute-all stops other peers only and is absent for non-chairs; token or room failure renders `LaneFailed` instead of a spinner (`flow-meeting-broadcast.md` §13).
- Bad `memberToken` / missing roster: handshake refuse → `connect_error` → linking **FAILED** gate (not endless PENDING), rendered with the shared `LaneFailed` card.
- Transport drop: `TransportError` → linking stays **PENDING** and reconnect continues.
- Two browsers on the same live meeting: a collaborative edit via session `actions` in one reaches the other; both settle with `synced` / linking READY.
- Forced server disconnect: the client reconnects and re-runs the sync handshake; edits made while disconnected survive because the client answers the server state vector.
- Meeting outside `WAITING_TO_START` / `STARTED`: a live write yields `meeting.live.error` `MEETING_NOT_LIVE` and clears `synced` (linking FAILED until a later successful sync).

## 12) Related

- `docs/platforms/website/ssr-boot-and-startup.md` — boot phases
- `docs/platforms/website/route-registry-contract.md` §3.1, §5.4 — nested params + organization-host route block
- `docs/platforms/backend/contracts/client-portal-http-website.md` — `/website` mount contract
- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — `/meeting` namespace, handshake auth, rooms, `meeting.live.*`
- `docs/platforms/website/flow-meeting-broadcast.md` — LiveKit client stage, room hook, media ceilings
- `docs/platforms/backend/contracts/livekit-media-plane.md` — join-token HTTP + helper + participant JWT cache
- `docs/platforms/backend/contracts/meeting-live-state.md` — CRDT document, `live_state` BLOB, deferred column apply
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket namespaces, rooms, live events
- `docs/platforms/backend/modules/nodejs-socket-library.md` §10 — `/meeting` child events
- `docs/invariants/website.md` W58
- `docs/invariants/backend.md` B24
- `.cursor/rules/organization-host-routing.mdc`
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/website-meeting-shell.mdc`
- `.cursor/rules/website-meeting-livekit-broadcast.mdc`
- `.cursor/rules/website-backend-policy-mirror.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/website-mpages-routes-params-contract.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
- `.cursor/skills/website-meeting-live-session/SKILL.md`
- `.cursor/skills/website-meeting-shell/SKILL.md`
- `.cursor/skills/website-meeting-broadcast/SKILL.md`

Change-set inventories: §10a–§10o (latest = live talk queue fields = §10o; prior = live agenda = §10n; LiveKit broadcast = §10m).
