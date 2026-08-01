# LiveKit Media Plane Contract (Current)

## 1) Scope

Shipped in this change set:

- dependency `livekit-server-sdk@2.17.0` (pinned) in `backend/`,
- server helper `LiveKitHelper` for room admin APIs + participant access-token minting,
- env keys documented in `backend/.env.example`: `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`,
- organization-host HTTP join-token path: `POST /website/custom/org/livekit_token` (`org_host` + member proof + live-registry `STARTED` gate + reuse-or-mint on `MeetingParticipant`) returning `{ token, url }`,
- website hook `useMeetingLiveKitToken` (discriminated union — `ready` carries both `token` and `url`),
- website client consumer: `livekit-client` `Room.connect` + broadcast A/V UI — contract lives in `../../website/flow-meeting-broadcast.md`.

Out of scope (not shipped):

- GraphQL mutations that mint tokens,
- setting `MeetingParticipant.attended_at` / `left_at` on join/leave,
- mapping `MeetingParticipant.type` → publish grants,
- LiveKit webhooks, egress/recording, SIP, agents,
- any LiveKit ORM model / room table / Zoom-style URL columns on `Meeting`,
- storing API secrets in docs or committed files (local `backend/.env` only; gitignored).

## 2) Domain purpose

LiveKit is the **media A/V plane** for meetings.

| Plane | Owner | Responsibility |
|---|---|---|
| Governance + durable session data | SQL (`Meeting` + children) | lifecycle, roster, agenda lines (`id` / `sort_order` / `subject`), decisions, votes, talk queue, attendance stamps |
| Live collaborative session state | Yjs `MeetingLiveMap` (`live_state`) | session mirrors + session-only fields (e.g. agenda `isLiveCreated`; talk `talkTurn` / `currentTalkMemberId`; decisions + nested `votes` from SQL) — see `meeting-live-state.md` |
| Media | LiveKit Room | audio/video tracks only |

LiveKit is **not** the source of agenda, votes, talk queue, or attendance truth. Durable agenda/talk/decision/vote authoring remains SQL; the live map may carry session fields and SQL mirrors (`meeting-live-state.md` §1.2 / §9).

Product naming: room name is derived from `Meeting.id` — not persisted as a meeting column.

## 3) Dependency

| Package | Version | Role |
|---|---|---|
| `livekit-server-sdk` | `2.17.0` | Official Node server SDK (`AccessToken`, `LiveKitAPI`, room service) |

Lockfile: `backend/yarn.lock`.

Browser counterpart is a **website** dependency (`livekit-client`, exact pin) — never imported in `backend/`, and the server SDK is never imported in `website/`. See `../../website/flow-meeting-broadcast.md` §3.

Install note: engine conflicts on some transitive packages may require `yarn add … --ignore-engines` in this repo’s Node version; version pin remains exact in `package.json`.

## 4) Environment

Keys (names only — values stay in local `.env`):

| Key | Purpose |
|---|---|
| `LIVEKIT_URL` | Project URL from LiveKit dashboard (prefer `wss://…`) |
| `LIVEKIT_API_KEY` | API key |
| `LIVEKIT_API_SECRET` | API secret |

Documented placeholders: `backend/.env.example`.

Runtime behavior:

- Helper trims env values.
- Incomplete env throws: `LiveKit env incomplete: LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET`.
- SDK Twirp layer rewrites `ws` → `http` for server API calls (`wss://` → `https://`).
- Token result returns a **client** URL via `clientUrl()` (`http(s)` → `ws(s)`).

## 5) Helper API

File: `backend/src/app/helpers/LiveKitHelper.ts`

Pattern: static helper (same family as payment helpers) — **not** an ORM model, **not** authz owner.

### 5.1 Room identity

```text
ejtmaa:meeting:{meetingId}
```

`roomName(meetingId)` rejects empty/whitespace `meetingId`.

### 5.2 Constants / room CreateRoom defaults

| Constant / default | Value | Meaning |
|---|---|---|
| `DEFAULT_ACCESS_TOKEN_TTL` | `"12h"` | Access token TTL from **mint time** when `options.ttl` omitted (product default; change this constant to retarget) |
| `createRoom` `emptyTimeout` | `10 * 60` seconds | Room closes if nobody joins |
| `createRoom` `departureTimeout` | `20` seconds | Grace after last participant leaves (reconnect window) |

Token TTL clock starts at `toJwt()` mint on the backend — not at room join and not at meeting create. Each member receives their own JWT; TTL is per token.

SDK default without our constant would be `6h`; Ejtmaa overrides to `12h`.

### 5.3 Methods

| Method | Behavior |
|---|---|
| `url()` | Raw trimmed `LIVEKIT_URL` |
| `clientUrl(host?)` | Normalize to `ws`/`wss` for `livekit-client` connect |
| `createRoom(meetingId, options?)` | `LiveKitAPI.room.createRoom` — optional; rooms also auto-create on first join per LiveKit docs |
| `getRoom(meetingId)` | `listRooms([name])` → first or `null` |
| `deleteRoom(meetingId, { ignoreMissing? })` | Deletes room; with `ignoreMissing` swallows `ServerError` 404 / `not_found` |
| `listParticipants(meetingId)` | Room participants |
| `removeParticipant(meetingId, identity)` | Disconnect participant (can rejoin with a new token) |
| `createAccessToken(options)` | `AccessToken` + `VideoGrant` (`roomJoin` + `room`) + `await toJwt()` → `{ url, token, roomName }` |

### 5.4 Access token options

Required: `meetingId`, non-empty `identity` (LiveKit: identity required when `roomJoin` is true).

Optional grant/fields forwarded when set: `name`, `ttl`, `metadata`, `attributes`, `canPublish`, `canSubscribe`, `canPublishData`, `canUpdateOwnMetadata`, `roomAdmin`, `hidden`.

If neither `canPublish` nor `canSubscribe` is set, LiveKit’s grant default applies (both publish and subscribe enabled).

### 5.5 Ownership boundary

| Helper owns | Requester / callers own |
|---|---|
| Room name derivation | Org + roster + meeting `STARTED` checks |
| Mint JWT / CreateRoom / DeleteRoom | When to mint (join media) |
| Env credential load | Setting `attended_at` / `left_at` |
| Client URL normalization | Mapping participant `type` → grants |

## 6) Join-token HTTP path (shipped)

### 6.1 Entry points

| Surface | Location |
|---|---|
| Route | `POST /website/custom/org/livekit_token` on `OrgCustomRouter` with per-route `middleware("org_host")` |
| Controller | `MeetingLiveKitTokenController` (meeting-domain name; lives under `custom/` next to `OrgStartController` — do **not** name it `OrgLiveKitTokenController`) |
| Website API key | `API.CUSTOM.ORG_LIVEKIT_TOKEN` → `/custom/org/livekit_token` |
| Website hook | `useMeetingLiveKitToken` → public `{ status, token, url }` (union: both values only when `ready`) |
| Website consumer | `useMeetingLiveKitRoom` → `Room.connect(url, token)` (`../../website/flow-meeting-broadcast.md` §5) |

### 6.2 Request / response

- Header: `organizationId` (resolved by `org_host` → `currentOrganization`).
- Body: `{ memberId, token, meetingId }` (`token` = `Member.access_token`).
- Success body: `{ token, url }` — the LiveKit JWT plus the **client** connect URL from `LiveKitHelper.clientUrl()` (`ws` / `wss`).

The URL travels with the token on purpose: the website has no LiveKit env key and must never hardcode a host, so the same response that authorizes a room also says which server to reach.

### 6.3 Controller steps (authz + mint)

1. Require body credentials (`memberId`, `token`, `meetingId`) else `NOT_VALID_CREDENTIAL`.
2. Prove `Member` by `id` + `access_token`.
3. Prove `Meeting` under `organization.get("id")` and member `organization_id` matches.
4. Prove `MeetingParticipant` roster row for `(meeting_id, member_id)`.
5. `STARTED` gate: `peekMeetingLiveDoc(meetingId)` only (**no create**). Missing in-process registry entry or `readLiveFields(doc).status !== "STARTED"` → `MEETING_NOT_LIVE`.
6. Reuse `MeetingParticipant.livekit_token` when `livekit_token_expires_at` is at least 6 hours ahead; otherwise mint via `LiveKitHelper.createAccessToken` (`identity` = member id string, default TTL `DEFAULT_ACCESS_TOKEN_TTL` / `12h`), persist token + `livekit_token_expires_at = now + 12 hours` (keep aligned with that constant).
7. Return `{ token, url }` — `url` is always recomputed from env via `LiveKitHelper.clientUrl()`, never persisted next to the cached JWT (a redeployed LiveKit host must not be pinned by an old roster row).

One LiveKit room per meeting; one stored JWT per roster member (never share across members).

### 6.4 Operational ceilings (intentional)

- **In-process peek:** `peekMeetingLiveDoc` only sees the live registry on **this Node process**. Multi-process / cold worker without the live doc → `MEETING_NOT_LIVE` even if SQL `Meeting.status` is `STARTED`.
- **Schema:** new participant columns require the repo’s normal ORM alter/sync locally — no checked-in migration file.
- **Generated types:** backend boot regenerates `backend/.types/controllers.ts` (gitignored) so `c.website.custom.MeetingLiveKitTokenController` type-checks.

### 6.5 Website hook contract

`website/src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts`:

| `status` | When |
|---|---|
| `idle` | `!can.enterLive`, and/or live session not `STARTED`, and/or missing `memberId` / `memberToken` / `meetingId` — **no** HTTP request |
| `pending` | Fetch in flight, or network error quiet-retry while still active |
| `ready` | Axios `READY` and **both** mapped `token` and `url` present |
| `error` | Axios `ERROR` with a response body (`isResType`) — business/auth failures; **no** quiet retry |

Public shape is a discriminated union: `ready` ⇒ `token: string` + `url: string`; every other status ⇒ both `null`. A response that omits `url` therefore stays `pending` (retry-eligible) instead of reaching `Room.connect` with a partial payload.

Quiet network retry: `config.skipNetworkToast: true` + `setTimeout` 3s while `active`. Declared on `AxiosRequestConfig` in `website/src/types/extends/global.ts`; honored in `website/src/resources/configs/axios.ts` reject middleware.

Consumed by `useMeetingLiveKitRoom` (the only caller), which owns the `Room` and the A/V surface. Depends on `useMeetingLiveSession().meeting.status` for the `STARTED` gate (product session, not SQL).

## 7) Failure modes

### 7.1 Helper

| Condition | Behavior |
|---|---|
| Missing/blank env | throw before API call |
| Blank `meetingId` / `identity` | throw validation `Error` |
| `deleteRoom` missing room, `ignoreMissing: true` | no-throw |
| Other LiveKit API errors | propagate (`ServerError` from SDK) |

### 7.2 Join-token HTTP (`MeetingLiveKitTokenController`)

| Condition | Thrown / behavior |
|---|---|
| Missing body fields / bad member token / meeting not in org / member org mismatch / not on roster | `NOT_VALID_CREDENTIAL` |
| No in-process live doc or live status ≠ `STARTED` | `MEETING_NOT_LIVE` (localized `ar`/`en` messages) |
| Incomplete LiveKit env during mint | helper throw (propagates as server error) |
| Missing / inactive organization header | `org_host` → `404` |

Website: `NOT_VALID_CREDENTIAL` still hits the global errors funnel (existing axios behavior). Network failures with `skipNetworkToast` do not toast; hook retries while `STARTED`.

## 8) Frontend / GQL

- Website: `useMeetingLiveKitToken` under `meeting/hooks/` feeds `useMeetingLiveKitRoom`, which connects the room and drives `MeetingLiveBroadcast`. Full client contract (peers, publish toggles, playback autoplay gate, cooperative mute-all, ceilings): `../../website/flow-meeting-broadcast.md`.
- No GQL surface for LiveKit tokens or `MeetingParticipant.livekit_*` columns.
- Publish grants are still LiveKit defaults for every roster member — `MeetingParticipant.type` is not mapped to `canPublish`, so participant-type media policy remains unenforced on both planes.

## 9) Traceability map (this change set)

### Backend (`backend/`)

| Path | Role | Section |
|---|---|---|
| `src/app/helpers/LiveKitHelper.ts` | Helper (already on main; media plane SoT) | §5 |
| `src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts` | Join-token HTTP (renamed from the prior `OrgLiveKitTokenController` draft); now returns `{ token, url }` | §6 |
| `src/app/http/controllers/website/custom/OrgLiveKitTokenController.ts` | **Deleted** — renamed to `MeetingLiveKitTokenController` | §6.1 |
| `src/app/http/routes/website.ts` | `POST /custom/org/livekit_token` + `org_host` | §6.1 |
| `src/app/orm/models/MeetingParticipant.ts` | `livekit_token` / `livekit_token_expires_at` | §6.3 |
| `src/resources/trans/ar/messages.ts` | `MEETING_NOT_LIVE` | §7.2 |
| `src/resources/trans/en/messages.ts` | `MEETING_NOT_LIVE` | §7.2 |
| `package.json` / `yarn.lock` | `livekit-server-sdk` pin `2.17.0` (already on main) | §3 |
| `.env.example` | Env key placeholders (already on main) | §4 |
| `.env` | Local secrets (gitignored) | excluded — never document values |
| `.types/controllers.ts` | Autoload controller map (gitignored; regenerated on boot) | §6.4 |

### Website (`website/`)

| Path | Role | Section |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts` | Fetch hook (`{ status, token, url }` union) | §6.5, §8 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitRoom.ts` | `Room.connect` owner + A/V state | §8; `../../website/flow-meeting-broadcast.md` §5 |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | Broadcast stage (probe UI removed) | §8; `../../website/flow-meeting-broadcast.md` §6 |
| `package.json` / `yarn.lock` | `livekit-client` exact pin | §3; `../../website/flow-meeting-broadcast.md` §3 |
| `src/resources/configs/axios/api.ts` | `CUSTOM.ORG_LIVEKIT_TOKEN` | §6.1 |
| `src/resources/configs/axios.ts` | Honor `skipNetworkToast` | §6.5 |
| `src/types/extends/global.ts` | `AxiosRequestConfig.skipNetworkToast` | §6.5 |
| `lib/tsconfig.tsbuildinfo` | Generated by `yarn type-check` | excluded from narrative |

### Root docs / governance

| Path | Role | Section |
|---|---|---|
| `docs/platforms/backend/contracts/livekit-media-plane.md` | This contract | all |
| `docs/platforms/backend/contracts/meeting-participant-domain.md` | Participant columns + GQL exclusion | related |
| `docs/platforms/backend/contracts/client-portal-http-website.md` | HTTP surface index | related |
| `docs/platforms/backend/contracts/http-and-requesters.md` | `org_host` first wired route | related |
| `docs/platforms/backend/contracts/meeting-domain.md` | Meeting media pointer | related |
| `docs/platforms/backend/modules/runtime-integrations.md` | Env + helper + join-token index | related |
| `docs/platforms/website/flow-meeting-broadcast.md` | Website client contract (room hook + stage + ceilings) | related |
| `docs/platforms/website/organization-host-routing.md` | Host limits + path map | related |
| `docs/platforms/website/README.md` | Website index row | related |
| `.cursor/rules/livekit-media-plane.mdc` | Durable media-plane invariants | governance |
| `.cursor/rules/website-meeting-livekit-broadcast.mdc` | Durable browser-client invariants | governance |
| `.cursor/skills/website-meeting-broadcast/SKILL.md` | Broadcast extension workflow | governance |
| `.cursor/rules/meeting-participant-roster.mdc` | Token-cache columns never on GQL | governance |
| `.cursor/skills/meeting-livekit-token/SKILL.md` | Cross-surface join-token workflow | governance |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `docs/platforms/backend/modules/runtime-integrations.md`
- `docs/platforms/backend/contracts/client-portal-http-website.md`
- `docs/platforms/website/organization-host-routing.md`
- `docs/platforms/website/flow-meeting-broadcast.md`
- `.cursor/skills/meeting-livekit-token/SKILL.md`
- `.cursor/skills/website-meeting-broadcast/SKILL.md`
- Official SDK: `livekit-server-sdk` AccessToken + `LiveKitAPI` room APIs
