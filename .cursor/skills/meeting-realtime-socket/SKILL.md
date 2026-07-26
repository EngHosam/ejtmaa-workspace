---
name: meeting-realtime-socket
description: >-
  Wires Meeting realtime on website useMeetingSocket and backend /meeting
  meeting.* controllers with MeetingAuthenticationIOMiddleware. Use when adding or
  fixing Meeting socket session/join/reconnect, meeting.join / meeting.leave events,
  MeetingConnectionIOController, MeetingJoinIOController, meeting-socket config, or
  Meeting page socket wiring.
---

# Meeting realtime socket (`meeting.*` on `/meeting`)

## When to Use

- Adding or changing `useMeetingSocket`, `meeting-socket.ts`, or the Meeting page socket wiring.
- Adding a new `/meeting` inbound meeting event (`meeting.join`, `meeting.leave`, …).
- Debugging join that fires once then stops, or join that does not return after a backend disconnect.

## Read first

- `docs/platforms/backend/contracts/meeting-realtime-socket.md`
- `docs/platforms/website/organization-host-routing.md` §5.1
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/nodejs-socket-handler-contract.mdc`
- `.cursor/skills/nodejs-socket-server-event/SKILL.md`

## Instructions

1. **Backend auth** stays on `MeetingAuthenticationIOMiddleware` (`meeting_auth`) for `/meeting`. Do not fold meeting proof into actor `AuthenticationIOMiddleware`. Controllers read via `current*` helpers and `MeetingSocketData`.
2. **Backend controllers** under `backend/src/app/socket/controllers/meeting/`, extending `SocketServerControllerBase`, `declare` socket with `MeetingSocketData`.
3. **Register in `io.ts` only:** controller aliases + `/meeting` route keys as dotted events (`"meeting.join": "meeting_join"`). Connection alias returns the absolute child set (today `["meeting.join"]`).
4. **Rooms:** join `Rooms.MEETING(meetingId)` from the connection controller only.
5. **Website config** at `website/src/resources/configs/meeting-socket.ts` (root sibling of `socket.ts`): `SOCKET_URL("meeting")` + handshake query. Do not nest this factory under `configs/socket/`.
6. **Website hook** at `components/meeting/hooks/useMeetingSocket.ts`:
   - Required inputs: `memberId`, `memberToken`, `meetingId`.
   - Owns `createSocketInstance` / `connect` / `disconnect` — never `getSocket`.
   - Returns: `{ connected, socket }`.
   - Emit `meeting.join` on every `connect`.
   - Manual `socket.connect()` on `io server disconnect`.
7. **Boot:** `prepareSocket` stays socket-free on an organization host; Meeting owns its own session.
8. **Do not mirror** inbound client→server events into `types/events.ts` / socket event registries.
9. **Verify** with existing scripts: `yarn type-check` in `backend/` and `website/`.

## Non-negotiable rules

- No meeting socket product hook under `ui/base/hooks`.
- No bare `join` event name — use `meeting.*`.
- No participant trust assumed from optional handshake `organizationId` alone.
- Meeting realtime lives only on `/meeting`.
