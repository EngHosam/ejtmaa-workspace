---
name: website-meeting-live-session
description: >-
  Ships or extends the website MeetingLiveSessionProvider surface
  (linking/can/actions/meeting/me/attendWindow) nested under MeetingLiveProvider,
  linking PENDING|READY|FAILED gate, and MeetingLinkingScreen under MeetingLayout.
  Use when adding Meeting product UI that must respect linking, wiring
  start/end/attend/left/setAgendaItemStatus or the talk-queue actions
  (raiseHand/lowerHand/giveTalkFloor/removeFromTalkQueue/endTalkFloor),
  attend-window clock ownership,
  or fixing the org-host meeting gate screen. For socket/Yjs transport, use skill
  meeting-realtime-socket instead. For LiveKit join-token fetch / mount,
  use skill meeting-livekit-token. For talk-queue ordering and UI rules,
  use skill website-meeting-talk-queue.
---

# Website Meeting live session

## When to Use

- Adding Meeting page/shell UI that depends on connection readiness or session capabilities.
- Changing `resolveMeetingLiveSession`, `MeetingLiveSessionProvider` / `useMeetingLiveSession`, `MeetingLinkingScreen`, or the linking branch in `MeetingLayout`.
- Wiring start/end/attend/left/setAgendaItemStatus or the talk-queue actions without screens touching the CRDT or `batch` directly.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.3 (surface), §5.1 (transport inputs), §5.4 (READY shell / drawer)
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/website-meeting-shell.mdc` (drawer tiles + READY header/page IA)
- `.cursor/rules/meeting-talk-queue.mdc` (talk-queue field, order, and role contract)
- `.cursor/rules/meeting-realtime-socket.mdc` (how `error` / connected / synced are set)
- `.cursor/skills/meeting-realtime-socket/SKILL.md` (socket + Yjs only)
- `.cursor/skills/website-meeting-shell/SKILL.md` (READY shell IA)

## Instructions

1. Mount `MeetingLiveProvider` → `MeetingLiveSessionProvider` → shell (layout-owned). Do not open a second Meeting socket. Do not fold session into the transport provider.
2. Prefer `useMeetingLiveSession()` for `linking` / `can` / `actions` / `meeting` / `me` / `attendWindow`. Resolve once in the session provider — screens only read context. Use `useMeetingLive` for transport flags or `batch` only when implementing a new session action — never from product screens.
3. Keep linking + capability math in `hooks/useMeetingLiveSession.tsx` (`MeetingLiveSessionState` = `linking` + `can`). Call `useMeetingAttendWindow` once in the session instance; expose `attendWindow`; pass `windowOpen` into resolve. Screens must not call the attend-window hook. Session also adds `actions`, `meeting`, `me`. Do not reintroduce nested `stages` or remapped meeting/me enums.
4. Linking gate: if `linking !== "READY"`, render `MeetingLinkingScreen` alone (no header/drawer/footer/children). The screen reads `linking` from `useMeetingLiveSession()` — do not pass it as a prop.
5. Gate chrome: PENDING → `Loadable`; FAILED → `FiAlertCircle` + `semanticColor.stateError`; show org logo **and** name when present; copy under `meetingLayout.linking`.
6. READY shell drawer/header IA belongs to skill `website-meeting-shell` — do not redefine tile sets here.
7. Actions: no-op when `!can.*`; write via internal `batch` only. Do not export `batch` on the session surface. Do not assign `meeting.*` / `me.*` from UI.
8. `setAgendaItemStatus(id, status)`: `can` = chair + live (`WAITING_TO_START` \| `STARTED`); write `meeting.agendaItems[id].status` only when the item exists. No SQL sync. Active agenda = `DISCUSSING` (no root pointer).
9. Talk-queue actions: `raiseHand` / `lowerHand` are non-chair + `enterLive` + live; `giveTalkFloor` / `removeFromTalkQueue` / `endTalkFloor` are chair + live. Selection helpers (`findQueueHead`, `nextTalkTurn`) stay **private** in this file so head selection has one definition; a multi-field write (grant = set floor + clear turn) goes in **one** `batch`. Any `can` input derived from the map (`queueHeadExists`) is computed once in the instance and passed into `resolveCan` — do not scan `participants` inside `resolveCan`. Rules: `.cursor/rules/meeting-talk-queue.mdc`.
10. Do not re-implement `connect_error` branching here — transport owns it; session only reads `error` / connected / synced.
11. Attend window: only via session `attendWindow` (hook called once in session; mirror math private). `can.attend` uses `windowOpen`. Init reads `attendWindow` for remaining-duration copy — never a second hook call. `can.enterLive` is navigation + LiveKit token gate (no action write).
12. Verify with `yarn type-check` in `website/`.
