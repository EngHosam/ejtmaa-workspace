# Meeting Realtime Socket (`/meeting`)

## Scope

The backend realtime surface a meeting attendee connects to: socket namespace `/meeting`, its handshake authentication, its connection room, and its inbound `meeting.*` events.

Website consumer (hook-owned session, handshake query): `docs/platforms/website/organization-host-routing.md` §5.1.
Framework mechanics (namespace registration, handler-array listener set): `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10.
Meeting domain model: `docs/platforms/backend/contracts/meeting-domain.md`.

## 1) Registration

`backend/src/resources/configs/socket/io.ts`:

- bootable middleware `"meeting_auth"` → `MeetingAuthenticationIOMiddleware`,
- controller `"meeting_connection"` → `controllers/meeting/MeetingConnectionIOController`,
- controller `"meeting_join"` → `controllers/meeting/MeetingJoinIOController`,
- namespace `/meeting` with `globalMiddlewares: ["meeting_auth"]`, `connection: "meeting_connection"`, child route `"meeting.join": "meeting_join"`.

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

Actor boundary: `AuthenticationIOMiddleware` (`auth`) serves `/customer` and `/supervisor` on a handshake `token`; its `SocketData` carries `token` / `user` / `customer` / `supervisor`. Meeting proof stays out of it.

Public organization identification (the HTTP `organizationId` header, the optional handshake `organizationId`) is never sufficient on its own — see invariant B24.

## 3) Controllers and rooms

| Controller | Event | Behavior |
|---|---|---|
| `MeetingConnectionIOController` | connection | joins `Rooms.MEETING(meetingId)` via `currentMeeting(socket, true)`; returns `["meeting.join"]` |
| `MeetingJoinIOController` | `meeting.join` | logs `socketId`, `organizationId` (via `currentOrganization`), `args`; returns `[]` |

The connection return value is the **absolute** listener set for the socket, so `MeetingJoinIOController` returning `[]` unbinds `meeting.join` until the next connection binds it again. Re-join after reconnect is owned by the website hook, which re-emits on every `connect`.

Room joins happen in the connection controller only.

`meeting.join` is inbound client → server: it is **not** an outbound notify event and is not mirrored into frontend event registries (`socket-event-mirroring.md` does not apply).

## 4) Constants

`backend/src/resources/consts/NotificationsConsts.ts`:

| Map | Key | Value |
|---|---|---|
| `Namespaces` | `MEETING` | `/meeting` |
| `FCM_Namespaces` | `MEETING` | `-meeting` |
| `Rooms` | `MEETING(meetingId)` | `meeting-{id}` |

## 5) Failure modes

| Scenario | Behavior |
|---|---|
| Handshake missing `memberId` / `memberToken` / `meetingId` | `NOT_VALID_CREDENTIAL` — connection refused |
| `memberToken` does not match `Member.access_token` | `NOT_VALID_CREDENTIAL` |
| Meeting belongs to another organization than the member | `NOT_VALID_CREDENTIAL` |
| Handshake `organizationId` disagrees with `meeting.organization_id` | `NOT_VALID_CREDENTIAL` |
| No `MeetingParticipant` roster row for the pair | `NOT_VALID_CREDENTIAL` |
| Organization not `ACTIVE` | `NOT_VALID_CREDENTIAL` |
| Second `meeting.join` on the same connection | not handled — the alias is unbound until reconnect (§3) |

## 6) Shipped limits (intentional)

1. `MeetingJoinIOController` is a log probe; no presence, roster broadcast, or state mutation happens on join.
2. Only `meeting.join` exists. Further events (`meeting.leave`, …) follow the same dotted naming and the connection controller's absolute return list.
3. `Member.access_token` is the whole meeting credential: rotating it revokes socket access, exposing it grants it (`member-domain.md`).

## 7) Traceability

| Path | Role |
|---|---|
| `src/resources/configs/socket/io.ts` | `/meeting` namespace, `meeting_auth`, meeting controllers, `meeting.join` child route |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | handshake proof, `MeetingSocketData`, `current*` helpers |
| `src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | actor handshake `token` for `/customer`, `/supervisor` |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | room join + bound listener set |
| `src/app/socket/controllers/meeting/MeetingJoinIOController.ts` | log-only `meeting.join` |
| `src/resources/consts/NotificationsConsts.ts` | `MEETING` namespace / FCM topic / room |

## 8) Related

- `docs/platforms/website/organization-host-routing.md` §5.1 — website session, handshake query, reconnect
- `docs/platforms/backend/modules/runtime-integrations.md` §5 — socket provider summary
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10 — handler contract and child events
- `docs/platforms/backend/contracts/meeting-domain.md` §7.1 — realtime surface of the domain
- `docs/platforms/backend/contracts/member-domain.md` — `access_token` as the meeting credential
- `docs/invariants/backend.md` B24
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
