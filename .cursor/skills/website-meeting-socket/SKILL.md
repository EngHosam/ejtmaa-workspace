---
name: website-meeting-socket
description: >-
  Wires organization-host Meeting realtime join on website useMeetingSocket and
  backend /org meeting.* child controllers. Use when adding or fixing Meeting
  socket join/reconnect, meeting.join / meeting.leave events, MeetingJoinIOController,
  or org-host Meeting page socket wiring.
---

# Website meeting socket (`meeting.*` on `/org`)

## When to Use

- Adding or changing `useMeetingSocket` or the Meeting page socket wiring.
- Adding a new `/org` inbound meeting event (`meeting.join`, `meeting.leave`, …).
- Debugging join that fires once then stops, or join that does not return after a backend disconnect.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.1, §6.3
- `docs/platforms/backend/modules/nodejs-socket-library.md` §7, §10
- `.cursor/rules/website-meeting-org-socket.mdc`
- `.cursor/rules/nodejs-socket-handler-contract.mdc`
- `.cursor/skills/nodejs-socket-server-event/SKILL.md`

## Instructions

1. **Backend controller** under `backend/src/app/socket/controllers/meeting/`, extending `SocketServerControllerBase`, `declare` socket/`SocketData`.
2. **Register in `io.ts` only:** controller alias (e.g. `meeting_join`) + `/org` route key as dotted event (`"meeting.join": "meeting_join"`).
3. **Connection bind:** `OrgConnectionIOController` must return every child alias that should be live after connect (absolute set). If a child handler returns `[]`, that alias is unbound until reconnect.
4. **Website hook** at `components/meeting/hooks/useMeetingSocket.ts`:
   - Required inputs: `memberId`, `memberToken`, `meetingId`.
   - Returns: `{ connected, socket }`.
   - Emit the event on ready+connected and on every `connect`.
   - Manual `socket.connect()` on `io server disconnect`.
5. **Do not mirror** inbound client→server events into `types/events.ts` / socket event registries.
6. **Verify** with existing scripts: `yarn type-check` in `backend/` and `website/`.

## Non-negotiable rules

- No meeting socket product hook under `ui/base/hooks`.
- No bare `join` event name — use `meeting.*`.
- No participant trust assumed from `socket.data.organization` alone.
