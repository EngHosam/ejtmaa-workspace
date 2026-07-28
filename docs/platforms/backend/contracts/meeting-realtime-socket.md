# Meeting Realtime Socket (`/meeting`)

## Scope

The backend realtime surface a meeting attendee connects to: socket namespace `/meeting`, its handshake authentication, its connection room, and the `meeting.live.*` events that carry the collaborative document.

State plane the events transport (Yjs document, `live_state` BLOB, registry): `docs/platforms/backend/contracts/meeting-live-state.md`.
Website consumer (layout-owned `MeetingLiveProvider` session, handshake query, SyncedStore): `docs/platforms/website/organization-host-routing.md` §5.1.
Framework mechanics (namespace registration, handler-array listener set): `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10.
Meeting domain model: `docs/platforms/backend/contracts/meeting-domain.md`.

## 1) Registration

`backend/src/resources/configs/socket/io.ts`:

- bootable middleware `"meeting_auth"` → `MeetingAuthenticationIOMiddleware`,
- controller `"meeting_connection"` → `controllers/meeting/MeetingConnectionIOController`,
- controller `"meeting_live_sync"` → `controllers/meeting/MeetingLiveSyncIOController`,
- controller `"meeting_live_update"` → `controllers/meeting/MeetingLiveUpdateIOController`,
- namespace `/meeting` with `globalMiddlewares: ["meeting_auth"]`, `connection: "meeting_connection"`, child routes `"meeting.live.sync": "meeting_live_sync"` and `"meeting.live.update": "meeting_live_update"`.

Driver options are namespace-agnostic: `transports: ["websocket"]`, no `maxHttpBufferSize` override (§6.3).

## 2) Handshake authentication

`backend/src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` runs before any `/meeting` controller:

1. Reads handshake `memberId`, `memberToken`, `meetingId` (required) and optional `organizationId` from `headers.x || query.x`.
2. Loads `Member` by `id` + `access_token` (`memberToken` = `Member.access_token`).
3. Loads `Meeting` by id; requires the same `organization_id` as the member.
4. If `organizationId` is present, requires it to equal `meeting.organization_id` (defense-in-depth).
5. Requires a `MeetingParticipant` roster row for `(meeting_id, member_id)`.
6. Loads the `ACTIVE` `Organization` for `meeting.organization_id`.
7. Stores plain keys on `socket.data`: `organization`, `meeting`, `member`, `participant` (`MeetingSocketData`).
8. Controllers read context through the exported helpers `currentOrganization` / `currentMeeting` / `currentMember` / `currentParticipant` `(socket, sure?)` — same pattern as the HTTP auth helpers. Missing value with `sure` → `NOT_VALID_CREDENTIAL`.

The `Meeting` row is loaded **once per connection** and reused by every event on that socket (§4, §6.2).

Actor boundary: `AuthenticationIOMiddleware` (`auth`) serves `/customer` and `/supervisor` on a handshake `token`; its `SocketData` carries `token` / `user` / `customer` / `supervisor`. Meeting proof stays out of it.

Public organization identification (the HTTP `organizationId` header, the optional handshake `organizationId`) is never sufficient on its own — see invariant B24.

## 3) Controllers and events

All three meeting controllers extend `backend/src/app/socket/controllers/meeting/MeetingIOControllerBase.ts`, which owns:

- the `MeetingSocketData` socket typing (declared once instead of per controller),
- `MEETING_BOUND_EVENTS = ["meeting.live.sync", "meeting.live.update"]` — the absolute listener set,
- `meetingBoundEvents({ only? , except? })` — the set, optionally narrowed,
- `rejectLive(code)` — emits `meeting.live.error` to the caller and returns the **unchanged** set, so a rejection never unbinds the session.

| Controller | Event | Behavior | Returns |
|---|---|---|---|
| `MeetingConnectionIOController` | connection | joins `Rooms.MEETING(meetingId)` via `currentMeeting(socket, true)` | `meetingBoundEvents()` |
| `MeetingLiveSyncIOController` | `meeting.live.sync` | loads the registry document and answers the caller with the diff it is missing plus the server state vector | `meetingBoundEvents()` |
| `MeetingLiveUpdateIOController` | `meeting.live.update` | validates, gates on status, applies to the document, broadcasts to the rest of the room | `meetingBoundEvents()` |

Because the handler-array return is absolute, every meeting handler returns the full set; nothing in this surface intentionally drops a listener.

Room joins happen in the connection controller only.

### 3.1 `meeting.live.sync` (handshake, both directions)

Inbound `{ stateVector?: string }` (base64 of `Y.encodeStateVector`, optional).

1. `getOrCreateMeetingLiveDoc(meetingId)`.
2. `Y.encodeStateAsUpdateV2(doc, clientVector)`; an unreadable vector falls back to the full state rather than failing the session.
3. Emits to the **caller only**: `{ update, stateVector }`, both base64.

The reply carries the server vector so the client can push back whatever the server is missing (as a normal `meeting.live.update`). Without that second leg, edits made while disconnected would be dropped on reconnect.

No status gate: reading the current state is allowed for any authenticated participant regardless of meeting status.

### 3.2 `meeting.live.update` (write + fan-out)

Inbound `{ update: string }` (base64 of a **V2** update).

| Step | Failure |
|---|---|
| `update` must be a non-empty string | `rejectLive("NOT_VALID")` |
| `meeting.get("status")` must be in `Meeting().LIVE_STATUSES` | `rejectLive("MEETING_NOT_LIVE")` |
| `Y.applyUpdateV2(doc, bytes)` | `rejectLive("NOT_VALID")` on malformed bytes |

On success the payload is re-emitted verbatim to `socket.to(Rooms.MEETING(meetingId))` — the sender is excluded because its own document already holds the change. The document mutation schedules the BLOB persist (`meeting-live-state.md` §2).

### 3.3 `meeting.live.error` (outbound rejection)

`{ code: MeetingLiveErrorCode }` with `"NOT_VALID" | "MEETING_NOT_LIVE"`, emitted to the offending socket only.

Client contract (`useMeetingLiveInstance` via `MeetingLiveProvider`; UI reads `useMeetingLive`): record the code, drop out of `synced` so the UI stops accepting edits, and wait for the next `connect` to re-run the sync handshake.

### 3.4 Mirroring

`meeting.live.*` events are **session** events of the `/meeting` namespace, consumed by the meeting hook's own listeners. They are not `OnCustomerEvent` / `OnUserEvent` notify events and must **not** be added to `website/src/types/events.ts` or the socket event registries (`socket-event-mirroring.md`).

## 4) Authorization

| Layer | Enforced by | Effect |
|---|---|---|
| Connection | `meeting_auth` handshake (§2) | no member token / roster row / ACTIVE org → no session at all |
| Read live state | none beyond the handshake | any participant may `meeting.live.sync` |
| Write live state | `Meeting().LIVE_STATUSES` in the update controller | only `WAITING_TO_START` and `STARTED` accept writes |

**Not gated yet:** `MeetingParticipant.type` (chairperson / member / viewer) plays no role — a `VIEWER` on the roster can write while the meeting is live. The participant-type gate lands with the meeting page UI; `currentParticipant` is already available for it.

The status read is the handshake snapshot (§2), not a fresh query — see §6.2.

## 5) Constants

`backend/src/resources/consts/NotificationsConsts.ts`:

| Map | Key | Value |
|---|---|---|
| `Namespaces` | `MEETING` | `/meeting` |
| `FCM_Namespaces` | `MEETING` | `-meeting` |
| `Rooms` | `MEETING(meetingId)` | `meeting-{id}` |

## 6) Failure modes

### 6.1 Handshake

| Scenario | Behavior |
|---|---|
| Handshake missing `memberId` / `memberToken` / `meetingId` | `NOT_VALID_CREDENTIAL` — connection refused |
| `memberToken` does not match `Member.access_token` | `NOT_VALID_CREDENTIAL` |
| Meeting belongs to another organization than the member | `NOT_VALID_CREDENTIAL` |
| Handshake `organizationId` disagrees with `meeting.organization_id` | `NOT_VALID_CREDENTIAL` |
| No `MeetingParticipant` roster row for the pair | `NOT_VALID_CREDENTIAL` |
| Organization not `ACTIVE` | `NOT_VALID_CREDENTIAL` |

### 6.2 Live events

| Scenario | Behavior |
|---|---|
| `meeting.live.update` with missing/empty `update` | `meeting.live.error` `NOT_VALID`; listeners stay bound |
| `meeting.live.update` on a meeting outside `LIVE_STATUSES` | `meeting.live.error` `MEETING_NOT_LIVE` |
| `meeting.live.update` with bytes that are not a V2 update | `meeting.live.error` `NOT_VALID` |
| `meeting.live.sync` with a corrupt `stateVector` | server silently answers with the full state (§3.1) |
| Meeting row deleted mid-session | registry load throws `"404"` through the errors funnel; no `meeting.live.error` is emitted |
| Meeting status changed outside the live session | the gate keeps using the handshake snapshot until the socket reconnects (`meeting-live-state.md` §7.3) |

### 6.3 Transport

Payloads are base64, which inflates the binary by roughly one third. The effective ceiling is the Socket.IO default `maxHttpBufferSize` of 1 MB per message, and no application-level cap or rate limit is applied to `meeting.live.update`. A single message above the limit closes the connection; the client reconnects and re-syncs.

## 7) Shipped limits (intentional)

1. Only `subject`, `type`, and `status` live in the document today. Presence, roster broadcast, and media stay out of this surface (media is LiveKit — `livekit-media-plane.md`).
2. No participant-type authorization (§4).
3. No application-level size cap or rate limit on live updates (§6.3).
4. `Member.access_token` is the whole meeting credential: rotating it revokes socket access, exposing it grants it (`member-domain.md`).
5. Single backend instance — the document registry is process memory (`meeting-live-state.md` §7.5).
6. `meeting.leave` and other lifecycle events do not exist; disconnect is the only exit.

## 8) Traceability

| Path | Role |
|---|---|
| `src/resources/configs/socket/io.ts` | `/meeting` namespace, `meeting_auth`, meeting controllers, `meeting.live.*` child routes |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | handshake proof, `MeetingSocketData`, `current*` helpers |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | actor handshake `token` for `/customer`, `/supervisor` |
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | shared socket typing, bound-event set, `rejectLive` |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | room join + bound listener set |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | sync handshake, server state vector, full-state fallback |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | payload validation, status gate, apply, room broadcast |
| `src/app/helpers/MeetingLiveDocHelper.ts` | document registry behind both live controllers (`meeting-live-state.md` §2) |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM topic / room |

Removed in this change set: `src/app/socket/controllers/meeting/MeetingJoinIOController.ts` (log-only `meeting.join` probe) together with its `meeting_join` alias and route key.

## 9) Related

- `docs/platforms/backend/contracts/meeting-live-state.md` — document, BLOB, registry, deferred column apply
- `docs/platforms/website/organization-host-routing.md` §5.1 — website session, handshake query, reconnect
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket provider summary
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10 — handler contract and child events
- `docs/platforms/backend/contracts/meeting-domain.md` §7.1 — realtime surface of the domain
- `docs/platforms/backend/contracts/member-domain.md` — `access_token` as the meeting credential
- `docs/invariants/backend.md` B24, B25
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
