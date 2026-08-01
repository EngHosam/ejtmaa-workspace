---
name: website-meeting-talk-queue
description: >-
  Ships or extends the organization-host Meeting talk queue: live-map-only
  talkTurn / currentTalkMemberId, session actions raiseHand / lowerHand /
  giveTalkFloor / removeFromTalkQueue / endTalkFloor, useMeetingTalkQueue rows,
  the chair MeetingTalkQueuePage + MeetingTalkQueueCard, the non-chair
  BroadcastHandButton with MeetingHandRaisedOffIcon, tile queue/floor badges,
  and the chair drawer talkQueue pulse. Use when adding or fixing a queue action,
  changing who may raise or grant, changing displayed queue numbers or the
  "ahead of me" count, wiring a durable TalkRecord sync, or debugging a queue
  whose head disagrees between the page and the grant action. For generic session
  wiring use website-meeting-live-session; for LiveKit media use
  website-meeting-broadcast; for drawer/page IA use website-meeting-shell.
---

# Website Meeting talk queue

## When to Use

- Adding, removing, or re-gating a talk-queue action (`raiseHand`, `lowerHand`, `giveTalkFloor`, `removeFromTalkQueue`, `endTalkFloor`).
- Changing queue ordering, the displayed place number, or the "people ahead of me" count.
- Editing `MeetingTalkQueuePage`, `MeetingTalkQueueCard`, `useMeetingTalkQueue`, `BroadcastHandButton`, `MeetingHandRaisedOffIcon`, or the chair `talkQueue` drawer pulse.
- Wiring the live queue to durable `TalkRecord` rows, or adding server-side enforcement.
- Debugging: the chair grants the floor to someone other than the row marked next; a member's place jumps; the hand button state disagrees with the map.

## Do Not Use For

- Session provider / linking mechanics — `website-meeting-live-session`.
- LiveKit room, tracks, publish, mute-all — `website-meeting-broadcast`.
- Drawer tile IA and page chrome tokens — `website-meeting-shell`.
- Socket / Yjs transport — `meeting-realtime-socket`.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.3 (`can` / `actions`), §5.5 (page + ordering rules), §8 limits 14–15
- `docs/platforms/website/flow-meeting-broadcast.md` §6.1, §6.3, §6.4 (tile badges, peer order, hand control)
- `.cursor/rules/meeting-talk-queue.mdc`
- `docs/platforms/backend/contracts/talk-record-domain.md` §1 (why no SQL row exists today)
- `docs/invariants/website.md` W61

## Instructions

1. **Confirm the state surface before writing code.** The feature is exactly `participants[*].talkTurn` and root `currentTalkMemberId` on the live map. Do not add a queue array, a position field, a count, or a second store. If the change needs durable data, that is a `TalkRecord` decision — escalate rather than inventing a live field.
2. **Add the capability first, in `resolveCan`.** New behavior = new `MeetingLiveCapability` + `canNone()` entry + `resolveCan` branch, with any map-derived input (like `queueHeadExists`) computed once in the session instance and passed in. Never scan `participants` inside `resolveCan`, and never gate UI on a raw `me?.type` check.
3. **Add the action beside it.** Actions live in `useMeetingLiveSessionInstance`, no-op when the matching `can.*` is false, and write inside one `batch`. A grant is a two-field write (`currentTalkMemberId` + clear that member's `talkTurn`) and must stay in the same batch so no client ever observes a member both queued and on the floor.
4. **Keep head selection single-sourced.** `findQueueHead` and `nextTalkTurn` are private helpers in `useMeetingLiveSession.tsx`. `useMeetingTalkQueue` must sort with the **same** comparator (`talkTurn` ascending, then lexicographic `id`). If you change one, change both in the same edit — a mismatch means the row the chair sees as next is not the row the action grants (W61).
5. **Never render `talkTurn`.** It is an allocation counter: it grows, skips, and is not the user's position. The page renders `place` (`index + 1` after sorting); the member's own control renders **people ahead** (lower turns, plus the floor holder when it is not me).
6. **Preserve FIFO.** `giveTalkFloor` takes no member argument and always targets the head; the card exposes Give floor only on `isHead && canGive`. `endTalkFloor` clears the floor without promoting anyone. `removeFromTalkQueue` clears one turn and leaves the floor alone. Any "priority", "move up", or "auto next" request is a product change, not an implementation detail.
7. **Respect the role split.** Non-chair raises/lowers only their own hand, from the broadcast action row. The chair administers from the `talkQueue` page and has no hand control. The header carries no talk control at all.
8. **Mirror state on all three surfaces or none.** Queue membership shows up as: the chair page row, the broadcast tile badge, and the chair drawer tile pulse. Adding a new state (for example "muted while speaking") means deciding its representation on each surface before shipping one of them.
9. **Copy:** page copy under `ui.layouts.meetingLayout.talkQueue`, stage copy under `…broadcast`, drawer aria under `…drawer` — ar + en with identical key sets. A control whose label names an **action** (the raised hand button) reuses that label as its accessible name; do not add a fixed noun aria key that contradicts what the user reads.
10. **Icons:** when a needed glyph does not exist in `react-icons`, add a local `IconType` component under `meeting/` with the `Meeting*` prefix (`MeetingHandRaisedOffIcon` = library path + `FiMicOff`-style slash) rather than approximating with an unrelated icon or stacking two elements.
11. **Say what is not enforced.** Every gate here is client-side; the socket accepts any roster member's write. Do not describe it as authorization in code comments, UI copy, or docs, and keep the ceiling rows in `organization-host-routing.md` §8 accurate.
12. **Verify:** `yarn type-check` in `website/`, then the two-browser pass in `organization-host-routing.md` §11 (raise from a member, watch the chair pulse and place numbers, grant, end, remove).
