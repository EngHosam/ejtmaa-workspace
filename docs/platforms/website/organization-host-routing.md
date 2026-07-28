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
| Layout | `website/src/app/ui/layouts/MeetingLayout.tsx` — `MeetingLiveProvider` → `MeetingLiveSessionProvider` + linking gate (§5.3) then organization-branded shell (header, drawer, footer — §5.4); mapped in `MyApp.getLayout()` under `case "MEETING"` |
| Page | `website/src/app/ui/pages/Meeting.tsx` — `Main()` returns `null` (no product UI yet); session still runs under `MeetingLiveProvider` in the layout |
| Live transport | `website/src/app/ui/components/meeting/hooks/useMeetingLive.tsx` — socket + Yjs + SyncedStore instance + provider + public context reader (product module under `components/meeting/`, **not** `ui/base/hooks`) |
| Live session surface | `meetingLiveSession.ts` + `MeetingLiveSessionProvider` / `useMeetingLiveSession` — `{ linking, can, actions, meeting, me }` (§5.3) |
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
- Root value **must** be an empty `{}`. Nesting e.g. `participants: {}` in the SyncedStore initializer throws at runtime; the CRDT fills nested maps after sync.
- The public hook reads `liveStore[MEETING_LIVE_MAP]` (typed `Partial<MeetingLiveMap>`), including `participants` once the server document has them.

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

Provider / public hook value: `{ connected, synced, error, meeting, batch }` (`MeetingLiveHookOp`). `meeting` is the reactive `Partial<MeetingLiveMap>` proxy; `batch(fn)` is `doc.transact(fn)` and is the transport write primitive. `MeetingLiveMap` (including `participants: Record<string, MeetingLiveParticipant>`) lives in the mirrored pair `website/src/types/meeting.ts` ↔ `backend/src/app/types/meeting.ts` with **no** GQL imports — see `.cursor/rules/meeting-live-map-mirror.mdc`.

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

### 5.3) Meeting live session surface (`linking` / `can` / `actions` / `meeting` / `me`)

Public product API for Meeting UI. Transport stays in §5.1 (`MeetingLiveProvider`). Session is a **nested** provider under it: `MeetingLiveSessionProvider` owns one resolve + `actions` for the whole tree; consumers read via `useMeetingLiveSession()`.

| Piece | File | Role |
|---|---|---|
| Pure state | `website/src/app/ui/components/meeting/meetingLiveSession.ts` | `resolveMeetingLiveSession(input)` → `MeetingLiveSessionState` (`linking` + `can` only) |
| Provider + hook | `website/src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | `MeetingLiveSessionProvider` (under `MeetingLiveProvider`) + public `useMeetingLiveSession()` context reader |
| Gate UI | `website/src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | Full-viewport PENDING / FAILED when linking is not READY |

#### Public shape

```ts
{
  linking: MeetingLiveLinking;           // "PENDING" | "READY" | "FAILED"
  can: MeetingLiveCapabilities;
  actions: MeetingLiveSessionActions;
  meeting: Partial<MeetingLiveMap>;     // read
  me: MeetingLiveParticipant | undefined; // read
}
```

`MeetingLiveLinking` is the **only** derived session enum. Meeting lifecycle and attendance are read from `meeting.status` and `me.attendedAt` / `me.leftAt` — no remapped stage enums.

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

When `linking !== "READY"`, every `can.*` = `false`. The session value still includes `meeting` / `me` proxies; the shell does not mount product chrome until READY.

#### Capabilities (`can`)

Computed only when linking is READY:

| Key | True when |
|---|---|
| `startMeeting` | `meType === "CHAIRPERSON"` and `status === "WAITING_TO_START"` |
| `endMeeting` | chairperson and `status === "STARTED"` |
| `attend` | any `meType`, status `WAITING_TO_START` or `STARTED`, no `attendedAt`, no `leftAt` |
| `left` | any `meType`, status `WAITING_TO_START` or `STARTED`, has `attendedAt`, no `leftAt` |

These are **client session gates** for the hook's actions. They do **not** replace the backend write gate (`MEETING_LIVE_STATUSES` on `meeting.live.update`) and do **not** yet enforce participant type on the server (§8).

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
| `FAILED` | Same identity, `FiAlertCircle` + `semanticColor.stateError`, fixed `failedMessage` copy |

i18n under `ui.layouts.meetingLayout.linking`: `logoAria`, `pendingStatus`, `failedMessage` (ar + en).

### 5.4) Meeting shell (organization branding)

`MEETING` is the only layout that renders tenant branding. The branding source is the `organizationHost` slice (§3) — the same public payload `org/start` returns — so the shell needs no extra request.

#### `useOrganization`

`website/src/app/ui/components/meeting/hooks/useOrganization.ts` is a product hook under `components/meeting/hooks/`, not `ui/base/hooks` (same placement rule as `useMeetingLive`).

- Reads `state.organizationHost` through `useSector` and returns `null` when `id` is absent, so a consumer cannot render tenant chrome on a non-organization host.
- Returns `{ id, name, description, logo_url, colors }`. Raw `primary_color` / `secondary_color` are **not** re-exported — a consumer must go through `colors`.
- `colors` is an `OrganizationColors` map whose keys mirror `semanticColor` naming, so a component reads `colors.cardBackground` exactly like it would read `semanticColor.cardBackground`.
- Seeds resolve with the installed `color` package: `primary_color` → brand, `secondary_color` → accent. A `null`, empty, or unparseable value falls back to `BrandColors.navy` / `BrandColors.orange`. The whole map is `useMemo`-ed on the six store fields.
- `defaultOrganizationColors()` builds the same map from the two `BrandColors` fallbacks; every consumer uses `organization?.colors ?? defaultOrganizationColors()` so the shell renders before the slice hydrates.

**Token split (non-negotiable).** Shell/neutral keys are assigned the `semanticColor.*` token itself, never a copied literal — `ColorType` accepts a `ThemeMapPath` and `getColor` resolves it per scheme, so these follow `theme.ts` automatically. Only the eight brand keys are computed from the seeds:

| Group | Keys | Source |
|---|---|---|
| Shell (16) | `pageBackground`, `headerBackground`, `footerBackground`, `drawerBackground`, `cardBackground`, `inputBackground`, `inputBorder`, `navigationBorder`, `footerBorder`, `subtleDivider`, `backdrop`, `textPrimary`, `textSecondary`, `textTertiary`, `iconPrimary`, `iconSecondary` | `semanticColor.<same key>` |
| Brand (8) | `primaryActionBackground`, `accentActionBackground`, `primaryActionText`, `accentActionText`, `sectionBrandBackground`, `sectionAccentBackground`, `textBrand`, `textAccent` | computed from the org seeds |

Copying a `ThemeMap` leaf into the shell group is a defect: `yarn type-check` validates a token path but cannot detect a stale hex, so a copy drifts silently on the next `theme.ts` edit. `primaryActionText` / `accentActionText` pick white or ink by seed luminance, so an organization may choose a light primary without losing contrast. Authority: `docs/invariants/website.md` W43.

Five of the brand keys — `accentActionText`, `sectionBrandBackground`, `sectionAccentBackground`, `textBrand`, `textAccent` — have no consumer in the shipped shell. They are a declared reserve for descendant meeting screens, not dead code; the YAGNI clause in W43 applies to `semanticColor` additions, not to this map's brand half, which is generated from the same two seeds regardless.

#### Shell components

| Component | Shipped behavior |
|---|---|
| `meeting/MeetingHeader.tsx` | Menu button (shared `HeaderIconButton`) + org logo, or the org name clamped to one line when there is no logo; for non-chair `me`, a disabled request-to-speak control (mic + `requestTalk` / `requestTalkAria`). A 2px rail in `colors.accentActionBackground` sits on the bottom edge. `fixed` prop switches between the in-flow desktop bar and the mobile `Fixed` bar at `zIndex.header`. |
| `meeting/MeetingFooter.tsx` | Rights line only: `© <year> <platform name> — <rights>`. The name is the **platform** (`ui.layouts.mainLayout.footerTitle`), not the organization. |
| `meeting/MeetingDrawerPanel.tsx` | Shared panel body for both breakpoints: title row (close when `showClose`), org identity card (logo and/or name), role-based 2-column tile grid (`live` full-row), pinned appearance/language row (`ThemeModeSwitch` + `LanguageSwitch`, both `compact`). |
| `meeting/MeetingDrawerOverlay.tsx` | Mobile portal overlay following the `CustomerDrawer` pattern: `createPortal` to `body`, `zIndex.modals`, blurred backdrop, RTL-aware slide-in, `no-scroll-drawer` body lock, 220ms unmount delay. Exported **without** `withMemo` — it reads `router.isRTL`. |

The button chrome is `website/src/app/ui/components/HeaderIconButton.tsx` — a top-level shared component (`component-structure.md` §3), consumed by both `CustomerHeader` and `MeetingHeader`. It owns its own tokens (`inputBackground`, `inputBorder`, `iconPrimary`, `semanticDims.control.iconButtonSize`, `semanticDims.card.radius`) and exposes no color props, so the two headers cannot drift.

The glyph is the shared `DrawerMenuIcon`. Its leading bar is themeable through the optional `accentClr` prop, defaulting to `semanticColor.accentActionBackground`; the two horizontal bars are always `semanticColor.iconPrimary`. `MeetingHeader` passes `semanticColor.iconPrimary` so all three bars read as one content-colored mark (dark in light scheme, near-white in dark). `CustomerHeader` passes nothing and keeps the accent bar.

**Drawer tiles.** Role-based from `useMeetingLiveSession().me?.type` (local `DrawerGridItemDef[]`). The READY shell (header/drawer) mounts only after `linking === "READY"`. Chairperson (`CHAIRPERSON`): `live` (`FiVideo`), `talkQueue` (`FiMic`), `attendance` (`FiClipboard`), `agenda` (`FiList`), `decisionsAndVote` (`FiCheckSquare`). Member and viewer: `live`, `agenda`, `decisionsAndVote` only. No `participants` tile. The `live` tile is full-row (`wide` → `gridColumn: 1 / -1`) so it spans both columns of the 2-column grid; other tiles stay single-cell. Every tile is passed `available={false}` (dimmed `opc 0.55`, `disabled` + `aria-disabled`, no click handler or route). Tile chrome mirrors `CustomerDrawer`'s `DrawerGridItem`: `minH 7.2`, a `3.1`-square icon well in `colors.primaryActionBackground` with the glyph in `colors.primaryActionText`, and a `smallAction` bold label clamped to two lines. Governance: `.cursor/rules/website-meeting-shell.mdc`, skill `website-meeting-shell`.

**Header request-to-speak.** When `me` exists and `me.type !== "CHAIRPERSON"`, `MeetingHeader` shows a disabled control: mic + visible `requestTalk` label; accessible name `requestTalkAria` (session-scoped, distinct from the label). MEMBER and VIEWER both get the control (`meeting-participant-domain.md` §8: request talking for all three types; chairperson uses drawer `talkQueue` instead). Shipped chrome is icon + label only — no switch track, no click handler. Header reads `me` from `useMeetingLiveSession()`.

#### Layout composition

`MeetingLayout` wraps both breakpoint trees in a single outer `MeetingLiveProvider` → `MeetingLiveSessionProvider`. Inside, `MeetingShell` runs the linking gate (§5.3) **before** picking desktop vs mobile chrome: when `linking !== "READY"`, only `MeetingLinkingScreen` mounts. When READY, the shell picks one of two trees from a `matchMedia(min-width: SW.min_lg)` effect (same shape as `MainLayout`). `children` is rendered once per READY tree.

- **Desktop:** a `Row` of [drawer column | `Col`(header, content, footer)]. The drawer column is **in flow** (not an overlay) and `position: sticky; top: 0` at `h/maxH: 100vh`, so it stays viewport-tall while the page scrolls and never covers the footer. Its width animates between `semanticDims.shell.drawerWidth` and `0`, with `pointerEvents: none` while collapsed.
- **Mobile:** the `CustomerMainLayout` shape — `MeetingHeader fixed`, content `minH: 100vh` with a `paddingTop` matching `Dims.headerHeight` / `Dims.mobileHeaderHeight`, and the portal overlay.

The panel is capped at `maxH: 100vh` with `minH: 0`; only the tile grid scrolls (`Col` + `flx_1` + `minH={0}` + `customScroll`), which keeps the appearance/language row visible at the bottom instead of pushing it below the fold as the tile list grows.

**i18n.** `ui.layouts.meetingLayout` in both `ar.ts` and `en.ts`, with identical key sets:

| Group | Keys |
|---|---|
| `header` | `menu`, `logoAria`, `requestTalk`, `requestTalkAria` |
| `footer` | `rights` |
| `linking` | `logoAria`, `pendingStatus`, `failedMessage` |
| `drawer` | `title`, `closeAriaLabel`, `logoAria`, `itemLive`, `itemTalkQueue`, `itemAttendance`, `itemAgenda`, `itemDecisionsAndVote`, `utilityPrefs` |

`MeetingFooter` additionally reads `ui.layouts.mainLayout.footerTitle` for the platform name; it has no key of its own for it.

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

1. **`org_host` is registered but unwired.** No HTTP route consumes `currentOrganization` yet; it is attached when organization-scoped endpoints land.
2. **`Meeting` page body is empty.** `Meeting.tsx` `Main()` returns `null`. Layout mounts the live session + linking gate; after READY, the branded shell mounts with empty page children — no meeting product sections yet.
3. **Collaborative live map fields** — `subject`, `type`, `status`, `participants`. Everything else on a meeting still goes through the customer GQL/requester path, and the live values are not reflected back onto the SQL columns yet (`../backend/contracts/meeting-live-state.md` §6).
4. **Non-production organization resolution ignores the request body** and always uses `TEST_ORGANIZATION_ID`, so local runs exercise a single organization.
5. **Handshake values travel primarily on the Socket.IO query** built in `meeting-socket.ts`. Header names are also read on the server (`headers.x || query.x`), but Node lowercases headers; do not rely on camelCase header-only delivery.
6. **`meeting.live.*` is not mirrored** into frontend event registries — it is a namespace session protocol. Outbound meeting notify events, when added, still follow `socket-event-mirroring.md`.
7. **Organization-host pages other than `Meeting` have no realtime channel.** A tenant-wide org socket is not part of the shipped surface.
8. **Server still has no participant-type write gate.** Any authenticated roster member can push live updates while status is live (`../backend/contracts/meeting-realtime-socket.md` §4). Website `can.startMeeting` / `can.endMeeting` are chairperson-only **client** gates on `useMeetingLiveSession` actions; they are not enforced by the socket controllers yet.
9. **Every meeting drawer tile is disabled** (§5.4). Role-based placeholders (chair: live / talk queue / attendance / agenda / decisions&vote; member+viewer: live / agenda / decisions&vote) have no route and no handler; the icon-well hover treatment is unreachable until tiles get targets. Header request-to-speak for MEMBER and VIEWER is likewise disabled.
10. **The meeting shell picks its READY tree in JavaScript**, so SSR always emits the mobile tree and a desktop client swaps after hydration. While linking is not READY, only the gate screen renders (no drawer SSR flicker for that path). `drawerOpen` is one state shared by both breakpoints and is re-seeded on every breakpoint crossing, so a manually collapsed desktop drawer reopens after a resize across `SW.min_lg`.
11. **Non-transport `connect_error` maps to session code `"NOT_VALID"`** for UI purposes; the linking FAILED copy stays the generic `failedMessage` (no distinct credential string on screen yet).

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
| `src/app/ui/pages/Meeting.tsx` | empty page body (`Main` → `null`); live session owned by layout | §5 |
| `src/app/ui/components/meeting/hooks/useMeetingLive.tsx` | `useMeetingLiveInstance` + `MeetingLiveProvider` + public `useMeetingLive`; SyncedStore root `{ [MEETING_LIVE_MAP]: {} }` as `Partial<MeetingLiveMap>`; `/meeting` session; `connect_error` linking branch | §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveMe.ts` | current-member `participants` proxy (no clone); used by session for `me` / `can` / `actions` | §5.2 |
| `src/app/ui/components/meeting/meetingLiveSession.ts` | pure `resolveMeetingLiveSession` → `MeetingLiveSessionState` (`linking` + `can`) | §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | `MeetingLiveSessionProvider` + `useMeetingLiveSession` — `{ linking, can, actions, meeting, me }` | §5.3 |
| `src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | PENDING / FAILED gate UI via session `linking` (`Loadable` / `FiAlertCircle`) | §5.3 |
| `src/types/meeting.ts` | mirrored live map (`MeetingLiveMap`, `participants`, `MEETING_LIVE_*`; pair with `backend/src/app/types/meeting.ts`) | §5.1; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `package.json` | `yjs` + `@syncedstore/core` + `@syncedstore/react` exact pins | §5.1 |
| `src/app/ui/layouts/MeetingLayout.tsx` | `MEETING` layout — `MeetingLiveProvider` → `MeetingLiveSessionProvider`, linking gate, branded shell, responsive READY tree | §5, §5.1, §5.3, §5.4 |
| `src/app/ui/base/core/MyApp.tsx` | `case "MEETING"` → `MeetingLayout` | §5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | org branding hook; `OrganizationColors` shell/brand token split | §5.4 |
| `src/app/ui/components/meeting/MeetingHeader.tsx` | menu + org logo/name; disabled request-to-speak (`requestTalk` / `requestTalkAria`) for MEMBER and VIEWER; accent rail; `fixed` mobile bar | §5.4 |
| `src/app/ui/components/meeting/MeetingFooter.tsx` | platform rights line | §5.4 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | role-based tile grid (`wide` live); all tiles `available={false}`; prefs row | §5.4 |
| `src/app/ui/components/meeting/MeetingDrawerOverlay.tsx` | mobile portal overlay | §5.4 |
| `src/app/ui/components/HeaderIconButton.tsx` | top-level shared icon button; consumed by `CustomerHeader` + `MeetingHeader` | §5.4; `component-structure.md` §3 |
| `src/app/ui/components/customer/CustomerHeader.tsx` | consumes `HeaderIconButton` from the shared top level | §5.4 |
| `src/app/ui/components/DrawerMenuIcon.tsx` | optional `accentClr` prop; default is the accent bar | §5.4 |
| `src/resources/translations/ar.ts`, `src/resources/translations/en.ts` | `ui.layouts.meetingLayout` (`header` incl. `requestTalk`/`requestTalkAria`, `footer`, `linking`, `drawer` role tiles) | §5.3, §5.4 |
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
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | shared socket typing, bound-event set, `rejectLive` | `../backend/contracts/meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | join `Rooms.MEETING`; bind the live event set | `../backend/contracts/meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | sync handshake + server state vector | `../backend/contracts/meeting-realtime-socket.md` §3.1 |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | validation, status gate, apply, room broadcast | `../backend/contracts/meeting-realtime-socket.md` §3.2 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | private codec+seed (incl. nested `participants`) + registry + BLOB persistence | `../backend/contracts/meeting-live-state.md` §1.3, §2 |
| `src/app/types/meeting.ts` | mirrored `MeetingLiveMap` / `MEETING_LIVE_*` / participant unions (pair with `website/src/types/meeting.ts`) | §5.1; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `src/app/orm/models/Meeting.ts` | `live_state` column only | `../backend/contracts/meeting-live-state.md` §1.1 |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | `live_state` excluded from GQL attrs | `../backend/contracts/meeting-live-state.md` §4 |
| `src/resources/configs/socket/io.ts` | `/meeting` namespace + `meeting_auth` + meeting controllers | `../backend/contracts/meeting-realtime-socket.md` §1 |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM / room | `../backend/contracts/meeting-realtime-socket.md` §4 |
| `.env.example` | `TEST_ORGANIZATION_ID=1` | §9 |

### Governance

| Path | Role |
|---|---|
| `.cursor/rules/organization-host-routing.mdc` | host mode, route gate, transport identification |
| `.cursor/rules/meeting-realtime-socket.mdc` | Meeting session placement, hook contract, `connect_error`, backend pairing |
| `.cursor/rules/website-meeting-live-session.mdc` | `MeetingLiveSessionProvider` / `useMeetingLiveSession` linking/can/actions/meeting/me + linking gate UI |
| `.cursor/rules/website-meeting-shell.mdc` | READY shell drawer tile IA + header request-to-speak |
| `.cursor/rules/meeting-live-state.mdc` | CRDT document ownership, V2 codec, BLOB exposure |
| `.cursor/rules/meeting-live-map-mirror.mdc` | identical `MeetingLiveMap` files on backend ↔ website |
| `.cursor/rules/sequelize-include-by-association-name.mdc` | Sequelize `include` by association name (backend companion) |
| `.cursor/rules/nodejs-socket-namespace-registration.mdc` | namespace / controller / room registration |
| `.cursor/rules/nodejs-socket-handler-contract.mdc` | connection return = absolute listener set |
| `.cursor/rules/socket-event-mirroring.mdc` | outbound mirror scope |
| `.cursor/rules/website-semantic-color-token-discipline.mdc` | runtime per-tenant color maps — shell keys reference `semanticColor`, only brand keys are computed (§5.4) |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | Meeting realtime transport workflow |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | Meeting session surface + linking gate workflow |
| `.cursor/skills/website-meeting-shell/SKILL.md` | Meeting READY shell drawer/header IA workflow |
| `.cursor/skills/nodejs-socket-server-event/SKILL.md` | backend socket surface workflow |

## 10a) Change set inventory (flat MeetingLiveSession + nested session provider)

Current delivery for the product session surface: drop nested `stages` remaps; ship `{ linking, can, actions, meeting, me }` via `MeetingLiveSessionProvider` under `MeetingLiveProvider`. Path map for the whole org-host contract remains in §10.

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

## 11) Verification

- `yarn type-check` in `website/` and in `backend/`.
- Apex host: `/` boots through `API.CUSTOM.START`; `/meeting/...` renders `Error` `404`.
- Organization host: boot calls `org/start` and hydrates `organizationHost`; `/meeting/...` mounts `MEETING` layout (`MeetingLiveProvider` → `MeetingLiveSessionProvider`) + linking gate; after READY, branded shell with empty page body; `/customer/...` renders `Error` `404`.
- Socket: organization host opens **no** boot socket; `MeetingLayout` opens `/meeting` once via `MeetingLiveProvider` / `useMeetingLiveInstance`, joins `meeting-{id}`, and emits `meeting.live.sync`; apex authed customer still connects to `/customer`.
- Bad `memberToken` / missing roster: handshake refuse → `connect_error` → linking **FAILED** gate (not endless PENDING).
- Transport drop: `TransportError` → linking stays **PENDING** and reconnect continues.
- Two browsers on the same live meeting: a collaborative edit via session `actions` in one reaches the other; both settle with `synced` / linking READY.
- Forced server disconnect: the client reconnects and re-runs the sync handshake; edits made while disconnected survive because the client answers the server state vector.
- Meeting outside `WAITING_TO_START` / `STARTED`: a live write yields `meeting.live.error` `MEETING_NOT_LIVE` and clears `synced` (linking FAILED until a later successful sync).

## 12) Related

- `docs/platforms/website/ssr-boot-and-startup.md` — boot phases
- `docs/platforms/website/route-registry-contract.md` §3.1, §5.4 — nested params + organization-host route block
- `docs/platforms/backend/contracts/client-portal-http-website.md` — `/website` mount contract
- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — `/meeting` namespace, handshake auth, rooms, `meeting.live.*`
- `docs/platforms/backend/contracts/meeting-live-state.md` — CRDT document, `live_state` BLOB, deferred column apply
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket namespaces, rooms, live events
- `docs/platforms/backend/modules/nodejs-socket-library.md` §10 — `/meeting` child events
- `docs/invariants/website.md` W58
- `docs/invariants/backend.md` B24
- `.cursor/rules/organization-host-routing.mdc`
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/website-meeting-shell.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/website-mpages-routes-params-contract.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
- `.cursor/skills/website-meeting-live-session/SKILL.md`
- `.cursor/skills/website-meeting-shell/SKILL.md`
