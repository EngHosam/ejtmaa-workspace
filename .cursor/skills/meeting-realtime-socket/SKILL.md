---
name: meeting-realtime-socket
description: >-
  Wires Meeting realtime transport on website MeetingLiveProvider /
  useMeetingLiveInstance and backend /meeting meeting.live.* controllers with
  MeetingAuthenticationIOMiddleware and the Yjs live document. Use when adding
  or fixing socket session, sync/reconnect, connect_error vs meeting.live.error,
  MeetingIOControllerBase, live sync/update controllers, MeetingLiveDocHelper,
  live_state persistence, or meeting-socket config. For stages/can/actions or
  linking gate UI, use skill website-meeting-live-session instead.
---

# Meeting realtime socket (`meeting.live.*` on `/meeting`)

## When to Use

- Adding or changing `useMeetingLive` / `useMeetingLiveInstance` / `MeetingLiveProvider` / `useMeetingLiveMe`, `meeting-socket.ts`, or backend `/meeting` controllers.
- Adding a new `/meeting` event or a new field to the live document.
- Debugging edits that do not propagate, handshake FAILED vs transport retry, a rejected write, duplicate sockets, or state lost after a reconnect.
- Touching `live_state` persistence or the live document registry.
- **Not** for product stages/can/actions or `MeetingLinkingScreen` — use `.cursor/skills/website-meeting-live-session/SKILL.md`.

## Read first

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-live-state.md` — document, BLOB, registry, deferred column apply
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.2
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10
- `.cursor/rules/meeting-realtime-socket.mdc`
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/sequelize-include-by-association-name.mdc`
- `.cursor/rules/nodejs-socket-handler-contract.mdc`

## Instructions

1. **Backend auth** stays on `MeetingAuthenticationIOMiddleware` (`meeting_auth`) for `/meeting`. Do not fold meeting proof into actor `AuthenticationIOMiddleware`. Controllers read via `current*` helpers and `MeetingSocketData`.
2. **Backend controllers** under `backend/src/app/socket/controllers/meeting/`, extending `MeetingIOControllerBase` — it owns the socket typing, the bound-event tuple, and `rejectLive`. Do not re-`declare` the socket per controller.
3. **Register in `io.ts` only:** controller aliases + `/meeting` dotted route keys (`"meeting.live.sync": "meeting_live_sync"`). A new event is also added to `MEETING_BOUND_EVENTS`, otherwise the connection controller unbinds it.
4. **Every handler returns `this.meetingBoundEvents()`**, including rejection paths. Reject with `rejectLive(code)` so the client gets `meeting.live.error` while the listeners stay bound.
5. **Rooms:** join `Rooms.MEETING(meetingId)` from the connection controller only; broadcast with `socket.to(room)` so the sender is not echoed.
6. **Document access** goes through `getOrCreateMeetingLiveDoc(meetingId)`. Never construct a second `Y.Doc` for a meeting, never write `live_state` outside `MeetingLiveDocHelper`. Codec/seed stay private in that helper — not on `Meeting` ORM.
6b. **Roster seed includes** use association names (`include: [{ association: "member", required: true }]`) — see `.cursor/rules/sequelize-include-by-association-name.mdc`.
7. **Codec:** V2 on the BLOB, the sync reply, and the broadcast; convert local V1 doc events with `convertUpdateFormatV1ToV2` before emitting. Payloads travel base64.
8. **Gate writes** on `MEETING_LIVE_STATUSES` from `types/meeting.ts`. Reads are open to any authenticated participant.
9. **Website config** at `website/src/resources/configs/meeting-socket.ts` (root sibling of `socket.ts`): `SOCKET_URL("meeting")` + handshake query. Do not nest this factory under `configs/socket/`.
10. **Website live module** at `components/meeting/hooks/useMeetingLive.tsx`:
    - `useMeetingLiveInstance` owns the transport (private): required `memberId` / `memberToken` / `meetingId`; `createSocketInstance` / `connect` / `disconnect` — never `getSocket`, and no second socket hook beside it.
    - Live document fields use `MeetingLiveMap` from `website/src/types/meeting.ts` (mirrored with `backend/src/app/types/meeting.ts` — see `.cursor/rules/meeting-live-map-mirror.mdc`). Never type the SyncedStore map from GQL enums.
    - SyncedStore shape `{ [MEETING_LIVE_MAP]: Partial<MeetingLiveMap> }` with initializer `{ [MEETING_LIVE_MAP]: {} }` only — do not nest `participants: {}` in the initializer (throws). Read `liveStore[MEETING_LIVE_MAP]`.
    - Rebuild the store + doc bundle when `meetingId` changes, and pass `[store]` to `useSyncedStore`.
    - Emit `meeting.live.sync` on every `connect`; answer the server `stateVector` in the reply.
    - Apply remote updates with origin `"remote"` and skip that origin when emitting.
    - Surface `error` and clear `synced` on `meeting.live.error`.
    - Native `connect_error`: `TransportError` → keep retry (no `error`); otherwise `error = "NOT_VALID"`, disable reconnection, disconnect.
    - Manual `socket.connect()` on `io server disconnect`.
    - Return `{ connected, synced, error, meeting, batch }`; all writes go through `batch`.
    - `MeetingLiveProvider` calls the instance once (params from `useCurrentParams` for `Meeting`) and publishes that value.
    - Public `useMeetingLive()` reads context only — low-level transport/CRDT.
    - Current participant: `useMeetingLiveMe()` indexes `meeting.participants[memberId]`; never clone; field writes use `batch`.
11. **Mount once** in `MeetingLayout` with a single outer `<MeetingLiveProvider>`. Linking gate chrome is owned by the website-meeting-live-session skill.
12. **Live map mirror:** if `MeetingLiveMap` / participant fields / `MEETING_LIVE_*` change, update **both** `backend/src/app/types/meeting.ts` and `website/src/types/meeting.ts` identically in the same change; confirm with a file diff. Seed nested `participants` as per-id `Y.Map`s in `MeetingLiveDocHelper` (not plain objects).
13. **Boot:** `prepareSocket` stays socket-free on an organization host; Meeting owns its own session.
14. **Do not mirror** `meeting.live.*` into `types/events.ts` / socket event registries.
15. **Verify** with existing scripts: `yarn type-check` in `backend/` and `website/`. Functional check: two browsers on one live meeting, forced disconnect with an offline edit, bad token → `error` set, transport drop → no `error` + retry. Confirm only one `/meeting` socket per tab.

## Non-negotiable rules

- No second live-map type beside the mirrored `MeetingLiveMap` pair.
- No live codec/statics on `Meeting` ORM — helper only.
- No cloning `useMeetingLiveMe()` before collaborative writes.
- No meeting socket product module under `ui/base/hooks`.
- No second `useMeetingLiveInstance` / Meeting socket under the same layout tree.
- No bare `sync` / `update` event names — use `meeting.live.*`.
- No narrowed listener set on a rejection path.
- No participant trust assumed from optional handshake `organizationId` alone.
- No writing meeting SQL columns from a socket controller (`meeting-live-state.md` §6).
- Meeting realtime lives only on `/meeting`.
- No product `stages` / `can` / `actions` math inside the transport module.
