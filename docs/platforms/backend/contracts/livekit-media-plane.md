# LiveKit Media Plane Contract (Current)

## 1) Scope

Shipped in this change set:

- dependency `livekit-server-sdk@2.17.0` (pinned) in `backend/`,
- server helper `LiveKitHelper` for room admin APIs + participant access-token minting,
- env keys documented in `backend/.env.example`: `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`.

Out of scope (not shipped):

- join / leave HTTP requesters or GraphQL mutations that mint tokens,
- website LiveKit client wiring,
- setting `MeetingParticipant.attended_at` / `left_at` on join/leave,
- mapping `MeetingParticipant.type` → publish grants in a requester,
- LiveKit webhooks, egress/recording, SIP, agents,
- any LiveKit ORM model / room table / Zoom-style URL columns on `Meeting`,
- storing API secrets in docs or committed files (local `backend/.env` only; gitignored).

## 2) Domain purpose

LiveKit is the **media A/V plane** for meetings.

| Plane | Owner | Responsibility |
|---|---|---|
| Governance + durable session data | SQL (`Meeting` + children) | lifecycle, roster, agenda, decisions, votes, talk queue, attendance stamps |
| Media | LiveKit Room | audio/video tracks only |

LiveKit is **not** the source of agenda, votes, talk queue, or attendance truth.

Product naming: room name is derived from `Meeting.id` — not persisted as a meeting column.

## 3) Dependency

| Package | Version | Role |
|---|---|---|
| `livekit-server-sdk` | `2.17.0` | Official Node server SDK (`AccessToken`, `LiveKitAPI`, room service) |

Lockfile: `backend/yarn.lock`.

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

## 6) Intended join flow (product; requester not shipped)

1. Meeting is `STARTED`; member is on `MeetingParticipant` roster.
2. Member requests join media (future requester).
3. Backend mints token via `createAccessToken` (`identity` = member id; grants from type).
4. Client connects with returned `url` + `token`.
5. Attendance SQL updates remain outside LiveKit.

One LiveKit room per meeting; one token per member join request (do not share JWTs across members).

## 7) Failure modes (helper)

| Condition | Behavior |
|---|---|
| Missing/blank env | throw before API call |
| Blank `meetingId` / `identity` | throw validation `Error` |
| `deleteRoom` missing room, `ignoreMissing: true` | no-throw |
| Other LiveKit API errors | propagate (`ServerError` from SDK) |

## 8) Frontend / GQL

No website or GQL surface in this change set.

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/helpers/LiveKitHelper.ts` | Helper source of truth | §5 |
| `backend/package.json` | `livekit-server-sdk` pin `2.17.0` | §3 |
| `backend/yarn.lock` | Lock resolution | §3 |
| `backend/.env.example` | Env key placeholders + comment | §4 |
| `backend/.env` | Local secrets (gitignored) | excluded — never document values |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | This contract | all |
| `.cursor/rules/livekit-media-plane.mdc` | Durable media-plane invariants | governance |
| `docs/platforms/backend/contracts/meeting-domain.md` | Meeting SQL + media pointer | related |
| `docs/platforms/backend/modules/runtime-integrations.md` | Env + helpers index | related |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `docs/platforms/backend/modules/runtime-integrations.md`
- Official SDK: `livekit-server-sdk` AccessToken + `LiveKitAPI` room APIs
