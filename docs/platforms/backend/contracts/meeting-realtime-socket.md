# Meeting Realtime Socket (`/meeting`)

## Scope

The backend realtime surface a meeting attendee connects to: socket namespace `/meeting`, its handshake authentication, its connection room, and the `meeting.live.*` events that carry the collaborative document.

State plane the events transport (Yjs document, `live_state` BLOB, registry): `docs/platforms/backend/contracts/meeting-live-state.md`.
Website consumer (layout-owned `MeetingLiveProvider` session, handshake query, SyncedStore, `useMeetingLiveSession` linking gate): `docs/platforms/website/organization-host-routing.md` §5.1, §5.3.
Framework mechanics (namespace registration, handler-array listener set): `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10.
Meeting domain model: `docs/platforms/backend/contracts/meeting-domain.md`.

## 1) Registration

`backend/src/resources/configs/socket/io.ts`:

- bootable middleware `"meeting_auth"` → `MeetingAuthenticationIOMiddleware`,
- controller `"meeting_connection"` → `controllers/meeting/MeetingConnectionIOController`,
- controller `"meeting_live_sync"` → `controllers/meeting/MeetingLiveSyncIOController`,
- controller `"meeting_live_update"` → `controllers/meeting/MeetingLiveUpdateIOController`,
- controller `"meeting_live_start"` → `controllers/meeting/MeetingLiveStartIOController`,
- controller `"meeting_live_complete"` → `controllers/meeting/MeetingLiveCompleteIOController`,
- controller `"meeting_disconnect"` → `controllers/meeting/MeetingDisconnectIOController`,
- namespace `/meeting` with `globalMiddlewares: ["meeting_auth"]`, `connection: "meeting_connection"`, child routes `"meeting.live.sync"`, `"meeting.live.update"`, `"meeting.live.start"`, `"meeting.live.complete"`, and `disconnect: ["meeting_disconnect", {}, "once"]`.

Driver options are namespace-agnostic: `transports: ["websocket"]`, no `maxHttpBufferSize` override (§6.3).

## 2) Handshake authentication

`backend/src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` runs before any `/meeting` controller:

1. Reads handshake `memberId`, `memberToken`, `meetingId` (required) and optional `organizationId` from `headers.x || query.x`.
2. Loads `Member` by `id` + `access_token` (`memberToken` = `Member.access_token`).
3. Loads `Meeting` by id; requires the same `organization_id` as the member.
4. If `organizationId` is present, requires it to equal `meeting.organization_id` (defense-in-depth).
5. Requires a `MeetingParticipant` roster row for `(meeting_id, member_id)`.
6. Loads the `ACTIVE` `Organization` for `meeting.organization_id`.
7. Loads the org `Customer` and requires `getCurrentSubscription` (else `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED`).
8. Stores plain keys on `socket.data`: `organization`, `meeting`, `member`, `participant` (`MeetingSocketData`).
9. Controllers read context through the exported helpers `currentOrganization` / `currentMeeting` / `currentMember` / `currentParticipant` `(socket, sure?)` — same pattern as the HTTP auth helpers. Missing value with `sure` → `NOT_VALID_CREDENTIAL`.

The `Meeting` row is loaded **once per connection** and reused by every event on that socket (§4, §6.2).

Actor boundary: `AuthenticationIOMiddleware` (`auth`) serves `/customer` and `/supervisor` on a handshake `token`; its `SocketData` carries `token` / `user` / `customer` / `supervisor`. Meeting proof stays out of it.

Public organization identification (the HTTP `organizationId` header, the optional handshake `organizationId`) is never sufficient on its own — see invariant B24.

## 3) Controllers and events

All meeting controllers extend `backend/src/app/socket/controllers/meeting/MeetingIOControllerBase.ts`, which owns:

- the `MeetingSocketData` socket typing (declared once instead of per controller),
- `MEETING_CHAIR_ONLY_EVENTS = ["meeting.live.start", "meeting.live.complete"]`,
- `MEETING_BOUND_EVENTS` composed as `sync` + `update` + `...MEETING_CHAIR_ONLY_EVENTS` + `disconnect`,
- `meetingBoundEvents({ only? , except? })` — chair sockets get the full set; non-chair omits chair-only events (`.some` membership, no string cast),
- `MeetingLiveErrorCode = "NOT_VALID" | "MEETING_NOT_LIVE" | "NOT_CHAIR"`,
- `rejectLive(code)` — emits `meeting.live.error` to the caller and returns the **role-appropriate** bound set, so a rejection never unbinds the session.

Chair detection uses `currentParticipant(this.socket)` from the middleware helpers — never raw `socket.data.participant`.

| Controller | Event | Behavior | Returns |
|---|---|---|---|
| `MeetingConnectionIOController` | connection | joins `Rooms.MEETING(meetingId)` via `currentMeeting(socket, true)` | `meetingBoundEvents()` |
| `MeetingLiveSyncIOController` | `meeting.live.sync` | loads the registry document and answers the caller with the diff it is missing plus the server state vector; maps `"MEETING_NOT_LIVE"` → `rejectLive` | `meetingBoundEvents()` |
| `MeetingLiveUpdateIOController` | `meeting.live.update` | validates, gates on status, applies to the document, broadcasts to the rest of the room; no SQL column writes; maps `"MEETING_NOT_LIVE"` → `rejectLive` | `meetingBoundEvents()` |
| `MeetingLiveStartIOController` | `meeting.live.start` | chair + `WAITING_TO_START` → SQL `STARTED`; bind socket meeting into registry entry when present; **no** Yjs mutate/rebroadcast | `meetingBoundEvents()` |
| `MeetingLiveCompleteIOController` | `meeting.live.complete` | chair + reload `STARTED` → `readLiveFields` + `completeMeetingLiveToSql`; **no** `meeting.live.completed` emit | `meetingBoundEvents()` |
| `MeetingDisconnectIOController` | `disconnect` (`once`) | Socket.IO native disconnect; temporary probe log of meeting/member/reason | `[]` (no listeners remain on a closed socket; post-handler sync is skipped when already disconnected) |

Because the handler-array return is absolute, every live meeting handler returns the full set for that socket's role; nothing in this surface intentionally drops a listener while the socket is open. `MeetingDisconnectIOController` returns `[]` because the socket is closing.

Room joins happen in the connection controller only.

### 3.1 `meeting.live.sync` (handshake, both directions)

Inbound `{ stateVector?: string }` (base64 of `Y.encodeStateVector`, optional).

1. `getOrCreateMeetingLiveDoc(meetingId)`; on `"MEETING_NOT_LIVE"` → `rejectLive("MEETING_NOT_LIVE")`.
2. `Y.encodeStateAsUpdateV2(doc, clientVector)`; an unreadable vector falls back to the full state rather than failing the session.
3. Emits to the **caller only**: `{ update, stateVector }`, both base64.

The reply carries the server vector so the client can push back whatever the server is missing (as a normal `meeting.live.update`). Without that second leg, edits made while disconnected would be dropped on reconnect.

Post-complete, sync is refused because `getOrCreate` will not re-seed a `COMPLETED` meeting.

### 3.2 `meeting.live.update` (write + fan-out)

Inbound `{ update: string }` (base64 of a **V2** update).

| Step | Failure |
|---|---|
| `update` must be a non-empty string | `rejectLive("NOT_VALID")` |
| `meeting.get("status")` must be in `MEETING_LIVE_STATUSES` | `rejectLive("MEETING_NOT_LIVE")` |
| `getOrCreate` throws `"MEETING_NOT_LIVE"` | `rejectLive("MEETING_NOT_LIVE")` |
| `Y.applyUpdateV2(doc, bytes)` | `rejectLive("NOT_VALID")` on malformed bytes |

On success the payload is re-emitted verbatim to `socket.to(Rooms.MEETING(meetingId))` — the sender is excluded because its own document already holds the change. The document mutation schedules the BLOB persist (`meeting-live-state.md` §2). **Must not** write meeting SQL columns.

### 3.3 `meeting.live.error` (outbound rejection)

`{ code: MeetingLiveErrorCode }` with `"NOT_VALID" | "MEETING_NOT_LIVE" | "NOT_CHAIR"`, emitted to the offending socket only.

This path is for **post-connect** rejects (bad update payload, status gate, apply failure, non-chair lifecycle). It is **not** how handshake auth failures surface.

Client contract (`useMeetingLiveInstance` via `MeetingLiveProvider`):

| Failure source | Client effect |
|---|---|
| `meeting.live.error` | store the code (incl. `NOT_CHAIR`), clear `synced` (listeners stay bound on the server) |
| Handshake refuse (`meeting_auth` → `NOT_VALID_CREDENTIAL`) | Socket.IO `connect_error` (no `meeting.live.error`). Website maps non-`TransportError` to session `error = "NOT_VALID"`, disables reconnection, and disconnects so UI linking becomes `FAILED`. Transport-only `connect_error` stays PENDING and keeps retrying. |

`MeetingLiveErrorCode` lives next to transport (`MeetingIOControllerBase` / `useMeetingLive`) — **not** in the mirrored `types/meeting.ts` CRDT contract.

Product UI reads linking via `useMeetingLiveSession().linking` under `MeetingLiveSessionProvider` (`organization-host-routing.md` §5.3).

### 3.4 `meeting.live.start` (chair lifecycle SQL)

No inbound payload.

1. `currentParticipant(...).get("type") === "CHAIRPERSON"` else `rejectLive("NOT_CHAIR")`.
2. Handshake meeting `status === "WAITING_TO_START"` else `rejectLive("MEETING_NOT_LIVE")`.
3. If a registry entry exists (`peekMeetingLiveDoc`), set `entry.meeting = meeting` (same instance as the socket row).
4. `meeting.update({ status: "STARTED" })`.
5. Do **not** mutate or broadcast the Yjs document — clients write live `status` through the existing `meeting.live.update` path.

### 3.5 `meeting.live.complete` (chair durable reflect)

No inbound payload. No outbound `meeting.live.completed`.

1. Chair gate (`NOT_CHAIR` otherwise).
2. `meeting.reload()` then `status === "STARTED"` else `rejectLive("MEETING_NOT_LIVE")`.
3. `getOrCreateMeetingLiveDoc`; on `"MEETING_NOT_LIVE"` → `rejectLive`.
4. `completeMeetingLiveToSql(meeting, readLiveFields(doc))` — owns `Meeting().transaction({transaction}, …)` + `afterCommit` destroy (`meeting-live-state.md` §6).
5. Return `meetingBoundEvents()`.

### 3.6 Mirroring

`meeting.live.*` events are **session** events of the `/meeting` namespace, consumed by the meeting hook's own listeners. They are not `OnCustomerEvent` / `OnUserEvent` notify events and must **not** be added to `website/src/types/events.ts` or the socket event registries (`socket-event-mirroring.md`).

## 4) Authorization

| Layer | Enforced by | Effect |
|---|---|---|
| Connection | `meeting_auth` handshake (§2) | no member token / roster row / ACTIVE org → no session at all |
| Read live state | handshake + live-status gate on `getOrCreate` | any participant may `meeting.live.sync` while the meeting is live; post-complete → `MEETING_NOT_LIVE` |
| Write live CRDT | `MEETING_LIVE_STATUSES` in the update controller | only `WAITING_TO_START` and `STARTED` accept CRDT writes |
| Lifecycle SQL | chair `MeetingParticipant.type === "CHAIRPERSON"` on start/complete | non-chair → `NOT_CHAIR`; events omitted from non-chair bound set |

**Still not gated on `meeting.live.update`:** participant type for ordinary CRDT field writes — a `VIEWER` on the roster can still push live updates while the meeting is live. Chair-only applies to **lifecycle** events only.

The update status read is the handshake snapshot (or rebound instance after start/complete on that socket) — see `meeting-live-state.md` §7.3.

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
| Organization customer missing active subscription (`getCurrentSubscription`) | `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` — connection refused |

Website: those refuses arrive as Socket.IO `connect_error` → linking `FAILED` (`organization-host-routing.md` §5.1, §5.3 / §7). They never become `meeting.live.error`.

### 6.2 Live events

| Scenario | Behavior |
|---|---|
| `meeting.live.update` with missing/empty `update` | `meeting.live.error` `NOT_VALID`; listeners stay bound |
| `meeting.live.update` on a meeting outside `MEETING_LIVE_STATUSES` | `meeting.live.error` `MEETING_NOT_LIVE` |
| `meeting.live.update` / `sync` / `complete` when `getOrCreate` throws `"MEETING_NOT_LIVE"` | `meeting.live.error` `MEETING_NOT_LIVE` |
| `meeting.live.update` with bytes that are not a V2 update | `meeting.live.error` `NOT_VALID` |
| `meeting.live.sync` with a corrupt `stateVector` | server silently answers with the full state (§3.1) |
| Non-chair `meeting.live.start` / `complete` | `meeting.live.error` `NOT_CHAIR` |
| Double complete / start when status wrong | `MEETING_NOT_LIVE` |
| Meeting row deleted mid-session | registry load throws `"404"` through the errors funnel; no `meeting.live.error` is emitted |
| Meeting status changed outside the live session | update gate may keep using the handshake snapshot until reconnect (`meeting-live-state.md` §7.3) |

### 6.3 Transport

Payloads are base64, which inflates the binary by roughly one third. The effective ceiling is the Socket.IO default `maxHttpBufferSize` of 1 MB per message, and no application-level cap or rate limit is applied to `meeting.live.update`. A single message above the limit closes the connection; the client reconnects and re-syncs.

## 7) Shipped limits (intentional)

1. Collaborative map fields stay document-owned during the session; durable SQL reflect runs on complete (`meeting-live-state.md` §6). Presence mutation writers (who flips `connectionStatus`) and media stay out of this surface for now (media is LiveKit — `livekit-media-plane.md`).
2. Ordinary CRDT updates still have no participant-type authorization (§4); lifecycle start/complete do.
3. No application-level size cap or rate limit on live updates (§6.3).
4. `Member.access_token` is the whole meeting credential: rotating it revokes socket access, exposing it grants it (`member-domain.md`).
5. Single backend instance — the document registry is process memory (`meeting-live-state.md` §7.5).
6. `meeting.leave` and other lifecycle events beyond start/complete do not exist; disconnect is the only exit path besides complete.

## 8) Traceability

| Path | Role |
|---|---|
| `src/resources/configs/socket/io.ts` | `/meeting` namespace, `meeting_auth`, meeting controllers, `meeting.live.*` child routes |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | handshake proof, `MeetingSocketData`, `current*` helpers |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | actor handshake `token` for `/customer`, `/supervisor` |
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | shared socket typing, bound-event set, `rejectLive` |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | room join + bound listener set |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | sync handshake, server state vector, full-state fallback |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | payload validation; write gate via `MEETING_LIVE_STATUSES`; apply; room broadcast; no SQL |
| `src/app/socket/controllers/meeting/MeetingLiveStartIOController.ts` | chair SQL `STARTED`; registry meeting bind |
| `src/app/socket/controllers/meeting/MeetingLiveCompleteIOController.ts` | chair complete → `completeMeetingLiveToSql` |
| `src/app/socket/controllers/meeting/MeetingDisconnectIOController.ts` | Socket.IO `disconnect` (`once`); temporary probe log |
| `src/app/helpers/MeetingLiveDocHelper.ts` | registry + `completeMeetingLiveToSql` (`meeting-live-state.md`) |
| `src/app/types/meeting.ts` | `MEETING_LIVE_STATUSES` / `MeetingLiveMap` mirror (`meeting-live-state.md` §1.2) |
| `src/app/helpers/MeetingLiveDocHelper.ts` | document registry behind both live controllers (`meeting-live-state.md` §1.3, §2) |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM topic / room |

## 9) Related

- `docs/platforms/backend/contracts/meeting-live-state.md` — document, BLOB, registry, deferred column apply
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.3 — website transport + session surface
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket provider summary
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10 — handler contract and child events
- `docs/platforms/backend/contracts/meeting-domain.md` §7.1 — realtime surface of the domain
- `docs/platforms/backend/contracts/member-domain.md` — `access_token` as the meeting credential
- `docs/invariants/backend.md` B24, B25
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
- `.cursor/skills/website-meeting-live-session/SKILL.md`
