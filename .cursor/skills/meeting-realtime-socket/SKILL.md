---
name: meeting-realtime-socket
description: >-
  Wires Meeting realtime on website useLiveMeeting and backend /meeting
  meeting.live.* controllers with MeetingAuthenticationIOMiddleware and the Yjs
  live document. Use when adding or fixing the Meeting socket session, sync or
  reconnect, meeting.live.sync / meeting.live.update / meeting.live.error,
  MeetingIOControllerBase, MeetingLiveSyncIOController, MeetingLiveUpdateIOController,
  MeetingLiveDocHelper, live_state persistence, meeting-socket config, or Meeting
  page wiring.
---

# Meeting realtime socket (`meeting.live.*` on `/meeting`)

## When to Use

- Adding or changing `useLiveMeeting`, `meeting-socket.ts`, the probe screen, or the Meeting page wiring.
- Adding a new `/meeting` event or a new field to the live document.
- Debugging edits that do not propagate, a session stuck on `syncing`, a rejected write, or state lost after a reconnect.
- Touching `live_state` persistence or the live document registry.

## Read first

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-live-state.md` — document, BLOB, registry, deferred column apply
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.2
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/nodejs-socket-handler-contract.mdc`

## Instructions

1. **Backend auth** stays on `MeetingAuthenticationIOMiddleware` (`meeting_auth`) for `/meeting`. Do not fold meeting proof into actor `AuthenticationIOMiddleware`. Controllers read via `current*` helpers and `MeetingSocketData`.
2. **Backend controllers** under `backend/src/app/socket/controllers/meeting/`, extending `MeetingIOControllerBase` — it owns the socket typing, the bound-event tuple, and `rejectLive`. Do not re-`declare` the socket per controller.
3. **Register in `io.ts` only:** controller aliases + `/meeting` dotted route keys (`"meeting.live.sync": "meeting_live_sync"`). A new event is also added to `MEETING_BOUND_EVENTS`, otherwise the connection controller unbinds it.
4. **Every handler returns `this.meetingBoundEvents()`**, including rejection paths. Reject with `rejectLive(code)` so the client gets `meeting.live.error` while the listeners stay bound.
5. **Rooms:** join `Rooms.MEETING(meetingId)` from the connection controller only; broadcast with `socket.to(room)` so the sender is not echoed.
6. **Document access** goes through `getOrCreateMeetingLiveDoc(meetingId)`. Never construct a second `Y.Doc` for a meeting, never write `live_state` outside `MeetingLiveDocHelper`.
7. **Codec:** V2 on the BLOB, the sync reply, and the broadcast; convert local V1 doc events with `convertUpdateFormatV1ToV2` before emitting. Payloads travel base64.
8. **Gate writes** on `Meeting().LIVE_STATUSES`. Reads are open to any authenticated participant.
9. **Website config** at `website/src/resources/configs/meeting-socket.ts` (root sibling of `socket.ts`): `SOCKET_URL("meeting")` + handshake query. Do not nest this factory under `configs/socket/`.
10. **Website hook** at `components/meeting/hooks/useLiveMeeting.ts`:
    - Required inputs: `memberId`, `memberToken`, `meetingId`.
    - Owns `createSocketInstance` / `connect` / `disconnect` — never `getSocket`, and no second socket hook beside it.
    - Rebuild the store + doc bundle when `meetingId` changes, and pass `[store]` to `useSyncedStore`.
    - Emit `meeting.live.sync` on every `connect`; answer the server `stateVector` in the reply.
    - Apply remote updates with origin `"remote"` and skip that origin when emitting.
    - Surface `error` and clear `synced` on `meeting.live.error`.
    - Manual `socket.connect()` on `io server disconnect`.
    - Return `{ connected, synced, error, meeting, batch }`; all writes go through `batch`.
11. **Boot:** `prepareSocket` stays socket-free on an organization host; Meeting owns its own session.
12. **Do not mirror** `meeting.live.*` into `types/events.ts` / socket event registries.
13. **Verify** with existing scripts: `yarn type-check` in `backend/` and `website/`. Functional check: two browsers on one live meeting, plus a forced disconnect with an offline edit.

## Non-negotiable rules

- No meeting socket product hook under `ui/base/hooks`.
- No bare `sync` / `update` event names — use `meeting.live.*`.
- No narrowed listener set on a rejection path.
- No participant trust assumed from optional handshake `organizationId` alone.
- No writing meeting SQL columns from a socket controller (`meeting-live-state.md` §6).
- Meeting realtime lives only on `/meeting`.
