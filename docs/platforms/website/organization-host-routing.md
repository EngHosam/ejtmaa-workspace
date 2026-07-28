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
- Meeting realtime is owned later by `LiveMeetingProvider` in `MeetingLayout` (§5.1),
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
| Layout | `website/src/app/ui/layouts/MeetingLayout.tsx` — organization-branded shell (header, drawer, footer — §5.3) plus a single `LiveMeetingProvider` wrap (§5.1); mapped in `MyApp.getLayout()` under `case "MEETING"` |
| Page | `website/src/app/ui/pages/Meeting.tsx` — renders `<LiveMeetingProbeScreen/>` with no credential props |
| Screen | `website/src/app/ui/components/meeting/LiveMeetingProbeScreen.tsx` — temporary probe UI (§5.2); reads live state via `useLiveMeeting()` |
| Live module | `website/src/app/ui/components/meeting/hooks/useLiveMeeting.tsx` (product module under `components/meeting/`, **not** `ui/base/hooks`) — session instance + provider + public context reader |

`Layout` in `website/src/types/extends/global.ts` includes the `"MEETING"` member; `PageRouteState` includes `orgHostOnly?: boolean`.

### 5.1) Live meeting session (`LiveMeetingProvider` + `/meeting`)

Meeting realtime is a **layout-owned** session on namespace `/meeting`, independent of the shared boot socket (§4). Exactly one socket, one Yjs document, and one reactive store exist for the Meeting shell tree. Consumers under the layout must not open a second session.

Dependencies: `yjs@13.6.27`, `@syncedstore/core@0.6.0`, `@syncedstore/react@0.6.0` (all exact-pinned, `yjs` matched to the backend copy).

Module: `website/src/app/ui/components/meeting/hooks/useLiveMeeting.tsx` (`.tsx` because it exports JSX for the provider). ThemeManager-shaped split:

| Symbol | Role |
|---|---|
| `useLiveMeetingInstance(data)` | Private session owner (socket + Yjs + SyncedStore). Called **only** inside `LiveMeetingProvider`. |
| `LiveMeetingProvider` | Reads `Meeting` route params via `useCurrentParams({ ident: "Meeting", mapParams: p => p })`, calls `useLiveMeetingInstance` once, publishes `LiveMeetingHookOp` on `LiveMeetingContext`. |
| `useLiveMeeting()` | Public default export — `useContext(LiveMeetingContext)` only (no args). Empty default when used outside the provider (same shape as `useThemeManager`). |

`MeetingLayout` wraps both desktop and mobile shell trees in **one** outer `<LiveMeetingProvider>` so the session is not duplicated per breakpoint branch.

**Document bundle** (inside `useLiveMeetingInstance`). `createLiveMeetingDocBundle()` builds `syncedStore({ meeting: {} })` plus its `getYjsDoc(store)`. The bundle is held in a ref keyed by `meetingId` and rebuilt when that id changes, so a route param change cannot keep editing the previous meeting's document. `useSyncedStore(store, [store])` receives the store in its dependency list — without it the memoized proxy would keep pointing at the old document.

**Session binding** (`bindLiveMeetingSession`), after the CSR gate (`useClientPreparing`) and the `memberId` / `memberToken` / `meetingId` guard:

1. `createSocketInstance(getMeetingSocketInstanceConfig(...))` with the handshake query from `meeting-socket.ts` (§3).
2. `doc.on("update")` → skip anything whose origin is `"remote"`, otherwise convert with `Y.convertUpdateFormatV1ToV2` and emit `meeting.live.update`. Local edits are always emitted, including before the first sync.
3. `connect` → `connected = true` and emit `meeting.live.sync` with the local `Y.encodeStateVector`.
4. `meeting.live.sync` reply → apply `update` when present, then answer the server's `stateVector` with `Y.encodeStateAsUpdateV2(doc, vector)` so anything edited offline is pushed back; clear `error` and set `synced`.
5. `meeting.live.update` → apply with origin `"remote"` so the change is not echoed back.
6. `meeting.live.error` → store the code and clear `synced`, which locks the UI until the next `connect` re-runs the handshake.
7. `disconnect` → clear `connected` and `synced`; on `io server disconnect` call `socket.connect()` (Socket.IO does not auto-reconnect after a server-forced drop).
8. Cleanup on unmount / deps change: `doc.off`, unregister the three listeners, `off` the native events, `disconnect`, reset state.

Provider / public hook value: `{ connected, synced, error, meeting, batch }` (`LiveMeetingHookOp`). `meeting` is the reactive `Partial<LiveMeetingFields>` proxy; `batch(fn)` is `doc.transact(fn)` and is the only sanctioned way to write. `LiveMeetingFields` types `type` and `status` with the generated `_MeetingTypeValue` / `_MeetingStatusValue`, so the live document cannot drift from the GraphQL enums.

Base64 helpers (`toBase64` / `fromBase64`) are local to the module because the payloads travel as base64 strings on the socket.

Backend pairing (`docs/platforms/backend/contracts/meeting-realtime-socket.md`): `meeting_auth` proves the handshake, the connection controller joins `Rooms.MEETING(meetingId)` and keeps both live events bound, and a rejected write answers `meeting.live.error` without unbinding anything.

**Not mirrored:** `meeting.live.*` are session events of this namespace. Do not add them to `website/src/types/events.ts` / socket event registries.

Authority: `.cursor/rules/meeting-realtime-socket.mdc`, `.cursor/rules/meeting-live-state.mdc`, `.cursor/skills/meeting-realtime-socket/SKILL.md`.

### 5.2) Probe screen (temporary)

`website/src/app/ui/components/meeting/LiveMeetingProbeScreen.tsx` is a **verification tool**, not the meeting product UI. It exists to prove the sync path end to end and is replaced when the real meeting page lands.

- Reads live state with arg-less `useLiveMeeting()` (context under `LiveMeetingProvider`). No credential props.
- Footer `meetingId` display uses `useCurrentParams({ ident: "Meeting", mapParams: p => p })` for the id only.
- Status line: `rejected · <code>` when `error` is set (in `semanticColor.stateError`), else `disconnected` / `connected · syncing` / `synced`.
- One `ProbeText` for `subject`, two `ProbeChoice` selects for `type` and `status`, whose options come from `Object.values(_MeetingTypeValue)` / `_MeetingStatusValue` — no hand-written label lists.
- Every field is disabled until `synced`, dimmed with the `opc` shorthand, and writes through `batch(() => { meeting.field = value })`.
- Chrome is a shared `fieldStyle` object on `ElementStyles.inputReset` with `semanticColor` / `semanticDims` tokens only; no literal colors or sizes.

It is deliberately **not** a registered form: no `Forms` entry, no `FormScreen`, no requester submit. The website form contract (`flow-form-foundation.md`) does not apply because nothing is submitted — edits go to the CRDT document.

### 5.3) Meeting shell (organization branding)

`MEETING` is the only layout that renders tenant branding. The branding source is the `organizationHost` slice (§3) — the same public payload `org/start` returns — so the shell needs no extra request.

#### `useOrganization`

`website/src/app/ui/components/meeting/hooks/useOrganization.ts` is a product hook under `components/meeting/hooks/`, not `ui/base/hooks` (same placement rule as `useLiveMeeting`).

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
| `meeting/MeetingHeader.tsx` | Menu button (shared `HeaderIconButton`) + org logo, or the org name clamped to one line when there is no logo. A 2px rail in `colors.accentActionBackground` sits on the bottom edge. `fixed` prop switches between the in-flow desktop bar and the mobile `Fixed` bar at `zIndex.header`. |
| `meeting/MeetingFooter.tsx` | Rights line only: `© <year> <platform name> — <rights>`. The name is the **platform** (`ui.layouts.mainLayout.footerTitle`), not the organization. |
| `meeting/MeetingDrawerPanel.tsx` | Shared panel body for both breakpoints: title row (with a close button only when `showClose`), org identity card (logo and/or name), a 2-column tile grid, and a pinned appearance/language row (`ThemeModeSwitch` + `LanguageSwitch`, both `compact`). |
| `meeting/MeetingDrawerOverlay.tsx` | Mobile portal overlay following the `CustomerDrawer` pattern: `createPortal` to `body`, `zIndex.modals`, blurred backdrop, RTL-aware slide-in, `no-scroll-drawer` body lock, 220ms unmount delay. Exported **without** `withMemo` — it reads `router.isRTL`. |

The button chrome is `website/src/app/ui/components/HeaderIconButton.tsx` — a top-level shared component (`component-structure.md` §3), consumed by both `CustomerHeader` and `MeetingHeader`. It owns its own tokens (`inputBackground`, `inputBorder`, `iconPrimary`, `semanticDims.control.iconButtonSize`, `semanticDims.card.radius`) and exposes no color props, so the two headers cannot drift.

The glyph is the shared `DrawerMenuIcon`. Its leading bar is themeable through the optional `accentClr` prop, defaulting to `semanticColor.accentActionBackground`; the two horizontal bars are always `semanticColor.iconPrimary`. `MeetingHeader` passes `semanticColor.iconPrimary` so all three bars read as one content-colored mark (dark in light scheme, near-white in dark). `CustomerHeader` passes nothing and keeps the accent bar.

**Drawer tiles.** Six tiles render from a local `DrawerGridItemDef[]`: `live` (`FiVideo`), `agenda` (`FiList`), `vote` (`FiCheckSquare`), `talkQueue` (`FiMic`), `decisions` (`FiFileText`), `participants` (`FiUsers`). Every tile is passed `available={false}`, so all six render dimmed (`opc 0.55`), carry `disabled` + `aria-disabled`, and have no click handler or route target. The tile chrome mirrors `CustomerDrawer`'s `DrawerGridItem`: `minH 7.2`, a `3.1`-square icon well in `colors.primaryActionBackground` with the glyph in `colors.primaryActionText`, and a `smallAction` bold label clamped to two lines.

#### Layout composition

`MeetingLayout` picks one of two trees from a `matchMedia(min-width: SW.min_lg)` effect, the same shape `MainLayout` uses. Both trees are wrapped in a single outer `LiveMeetingProvider`, so the `/meeting` socket session is owned by the layout once — not by the page and not once per breakpoint branch. `children` is rendered once per tree.

- **Desktop:** a `Row` of [drawer column | `Col`(header, content, footer)]. The drawer column is **in flow** (not an overlay) and `position: sticky; top: 0` at `h/maxH: 100vh`, so it stays viewport-tall while the page scrolls and never covers the footer. Its width animates between `semanticDims.shell.drawerWidth` and `0`, with `pointerEvents: none` while collapsed.
- **Mobile:** the `CustomerMainLayout` shape — `MeetingHeader fixed`, content `minH: 100vh` with a `paddingTop` matching `Dims.headerHeight` / `Dims.mobileHeaderHeight`, and the portal overlay.

The panel is capped at `maxH: 100vh` with `minH: 0`; only the tile grid scrolls (`Col` + `flx_1` + `minH={0}` + `customScroll`), which keeps the appearance/language row visible at the bottom instead of pushing it below the fold as the tile list grows.

**i18n.** `ui.layouts.meetingLayout` in both `ar.ts` and `en.ts`, with identical key sets:

| Group | Keys |
|---|---|
| `header` | `menu`, `logoAria` |
| `footer` | `rights` |
| `drawer` | `title`, `closeAriaLabel`, `logoAria`, `itemLive`, `itemAgenda`, `itemVote`, `itemTalkQueue`, `itemDecisions`, `itemParticipants`, `utilityPrefs` |

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
| Socket `/meeting` handshake missing ids, bad token, org mismatch, or no roster row | `NOT_VALID_CREDENTIAL` |
| Live write on a meeting that is not `WAITING_TO_START` / `STARTED` | `meeting.live.error` `MEETING_NOT_LIVE` → probe shows `rejected · MEETING_NOT_LIVE` and locks the fields |
| Malformed live payload | `meeting.live.error` `NOT_VALID` → same lock; the next `connect` re-syncs |
| Route params change to another meeting | `LiveMeetingProvider` re-runs `useLiveMeetingInstance` with the new ids; the instance drops the old document and store and opens a fresh session (§5.1) |

## 8) Known limits (shipped state, intentional)

1. **`org_host` is registered but unwired.** No HTTP route consumes `currentOrganization` yet; it is attached when organization-scoped endpoints land.
2. **`Meeting` UI is a sync probe only** (§5.2): status text plus three raw fields. No meeting content, media, agenda, or participant UX ships yet.
3. **Only three fields are collaborative** — `subject`, `type`, `status`. Everything else on a meeting still goes through the customer GQL/requester path, and the live values are not reflected back onto the SQL columns yet (`../backend/contracts/meeting-live-state.md` §6).
4. **Non-production organization resolution ignores the request body** and always uses `TEST_ORGANIZATION_ID`, so local runs exercise a single organization.
5. **Handshake values travel primarily on the Socket.IO query** built in `meeting-socket.ts`. Header names are also read on the server (`headers.x || query.x`), but Node lowercases headers; do not rely on camelCase header-only delivery.
6. **`meeting.live.*` is not mirrored** into frontend event registries — it is a namespace session protocol. Outbound meeting notify events, when added, still follow `socket-event-mirroring.md`.
7. **Organization-host pages other than `Meeting` have no realtime channel.** A tenant-wide org socket is not part of the shipped surface.
8. **No participant-type gate on the client or the server.** Any roster member of a live meeting can edit the document (`../backend/contracts/meeting-realtime-socket.md` §4).
9. **Every meeting drawer tile is disabled** (§5.3). The six tiles (live, agenda, vote, talk queue, decisions, participants) are placeholders with no route and no handler; the hover treatment on the icon well is therefore unreachable until the tiles get targets.
10. **The meeting shell picks its tree in JavaScript**, so SSR always emits the mobile tree and a desktop client swaps after hydration. `drawerOpen` is one state shared by both breakpoints and is re-seeded on every breakpoint crossing, so a manually collapsed desktop drawer reopens after a resize across `SW.min_lg`.

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
| `src/app/ui/pages/Meeting.tsx` | renders `<LiveMeetingProbeScreen/>` (no credential props) | §5, §5.2 |
| `src/app/ui/components/meeting/hooks/useLiveMeeting.tsx` | `useLiveMeetingInstance` + `LiveMeetingProvider` + public `useLiveMeeting`; `/meeting` session, Yjs, SyncedStore, `meeting.live.*` | §5.1 |
| `src/app/ui/components/meeting/hooks/useLiveMeeting.ts` | **deleted / renamed** to `.tsx` (JSX provider) | §5.1 |
| `src/app/ui/components/meeting/LiveMeetingProbeScreen.tsx` | temporary sync probe UI; consumes `useLiveMeeting()` | §5.2 |
| `src/app/ui/components/meeting/hooks/useMeetingSocket.ts` | **deleted** — socket-only hook absorbed into the live module; a separate session hook is now forbidden | §5.1 |
| `package.json` | `yjs` + `@syncedstore/core` + `@syncedstore/react` exact pins | §5.1 |
| `src/app/ui/layouts/MeetingLayout.tsx` | `MEETING` layout — branded shell, responsive tree, sticky desktop drawer, single `LiveMeetingProvider` wrap | §5, §5.1, §5.3 |
| `src/app/ui/base/core/MyApp.tsx` | `case "MEETING"` → `MeetingLayout` | §5 |
| `src/app/ui/components/meeting/hooks/useOrganization.ts` | org branding hook; `OrganizationColors` shell/brand token split | §5.3 |
| `src/app/ui/components/meeting/MeetingHeader.tsx` | menu button + org logo/name, accent rail, `fixed` mobile bar | §5.3 |
| `src/app/ui/components/meeting/MeetingFooter.tsx` | platform rights line | §5.3 |
| `src/app/ui/components/meeting/MeetingDrawerPanel.tsx` | shared drawer body, scrolling tile grid, pinned prefs row | §5.3 |
| `src/app/ui/components/meeting/MeetingDrawerOverlay.tsx` | mobile portal overlay | §5.3 |
| `src/app/ui/components/HeaderIconButton.tsx` | top-level shared icon button; consumed by `CustomerHeader` + `MeetingHeader` | §5.3; `component-structure.md` §3 |
| `src/app/ui/components/customer/CustomerHeader.tsx` | consumes `HeaderIconButton` from the shared top level | §5.3 |
| `src/app/ui/components/DrawerMenuIcon.tsx` | optional `accentClr` prop; default is the accent bar | §5.3 |
| `src/resources/translations/ar.ts`, `src/resources/translations/en.ts` | `ui.layouts.meetingLayout` (`header`, `footer`, `drawer`) | §5.3 |
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
| `src/app/helpers/MeetingLiveDocHelper.ts` | live document registry + BLOB persistence | `../backend/contracts/meeting-live-state.md` §2 |
| `src/app/orm/models/Meeting.ts` | `live_state` column + live document statics | `../backend/contracts/meeting-live-state.md` §1 |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | `live_state` excluded from GQL attrs | `../backend/contracts/meeting-live-state.md` §4 |
| `src/resources/configs/socket/io.ts` | `/meeting` namespace + `meeting_auth` + meeting controllers | `../backend/contracts/meeting-realtime-socket.md` §1 |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM / room | `../backend/contracts/meeting-realtime-socket.md` §4 |
| `.env.example` | `TEST_ORGANIZATION_ID=1` | §9 |

### Governance

| Path | Role |
|---|---|
| `.cursor/rules/organization-host-routing.mdc` | host mode, route gate, transport identification |
| `.cursor/rules/meeting-realtime-socket.mdc` | Meeting session placement, hook contract, backend pairing |
| `.cursor/rules/meeting-live-state.mdc` | CRDT document ownership, V2 codec, BLOB exposure |
| `.cursor/rules/nodejs-socket-namespace-registration.mdc` | namespace / controller / room registration |
| `.cursor/rules/nodejs-socket-handler-contract.mdc` | connection return = absolute listener set |
| `.cursor/rules/socket-event-mirroring.mdc` | outbound mirror scope |
| `.cursor/rules/website-semantic-color-token-discipline.mdc` | runtime per-tenant color maps — shell keys reference `semanticColor`, only brand keys are computed (§5.3) |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | Meeting realtime workflow |
| `.cursor/skills/nodejs-socket-server-event/SKILL.md` | backend socket surface workflow |

## 11) Verification

- `yarn type-check` in `website/` and in `backend/`.
- Apex host: `/` boots through `API.CUSTOM.START`; `/meeting/...` renders `Error` `404`.
- Organization host: boot calls `org/start` and hydrates `organizationHost`; `/meeting/...` renders the probe on the `MEETING` layout; `/customer/...` renders `Error` `404`.
- Socket: organization host opens **no** boot socket; `MeetingLayout` opens `/meeting` once via `LiveMeetingProvider` / `useLiveMeetingInstance`, joins `meeting-{id}`, and emits `meeting.live.sync`; apex authed customer still connects to `/customer`.
- Two browsers on the same live meeting: an edit in one reaches the other, and the status line reads `synced` on both.
- Forced server disconnect: the client reconnects and re-runs the sync handshake; edits made while disconnected survive because the client answers the server state vector.
- Meeting outside `WAITING_TO_START` / `STARTED`: editing shows `rejected · MEETING_NOT_LIVE` and the fields lock.

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
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/website-mpages-routes-params-contract.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
