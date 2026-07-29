---
name: website-meeting-live-session
description: >-
  Ships or extends the website MeetingLiveSessionProvider surface
  (linking/can/actions/meeting/me) nested under MeetingLiveProvider, linking
  PENDING|READY|FAILED gate, and MeetingLinkingScreen under MeetingLayout.
  Use when adding Meeting product UI that must respect linking, wiring
  start/end/attend/left actions, or fixing the org-host meeting gate screen.
  For socket/Yjs transport, use skill meeting-realtime-socket instead.
---

# Website Meeting live session

## When to Use

- Adding Meeting page/shell UI that depends on connection readiness or session capabilities.
- Changing `resolveMeetingLiveSession`, `MeetingLiveSessionProvider` / `useMeetingLiveSession`, `MeetingLinkingScreen`, or the linking branch in `MeetingLayout`.
- Wiring start/end/attend/left without screens touching the CRDT or `batch` directly.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.3 (surface), §5.1 (transport inputs), §5.4 (READY shell / drawer)
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/website-meeting-shell.mdc` (drawer tiles + header request-to-speak)
- `.cursor/rules/meeting-realtime-socket.mdc` (how `error` / connected / synced are set)
- `.cursor/skills/meeting-realtime-socket/SKILL.md` (socket + Yjs only)
- `.cursor/skills/website-meeting-shell/SKILL.md` (READY shell IA)

## Instructions

1. Mount `MeetingLiveProvider` → `MeetingLiveSessionProvider` → shell (layout-owned). Do not open a second Meeting socket. Do not fold session into the transport provider.
2. Prefer `useMeetingLiveSession()` for `linking` / `can` / `actions` and for reading `meeting` / `me`. Resolve once in the session provider — screens only read context. Use `useMeetingLive` for transport flags or `batch` only when implementing a new session action — never from product screens.
3. Keep linking + capability math in `meetingLiveSession.ts` (`MeetingLiveSessionState` = `linking` + `can`). Session instance adds `actions`, `meeting`, `me`. Do not reintroduce nested `stages` or remapped meeting/me enums.
4. Linking gate: if `linking !== "READY"`, render `MeetingLinkingScreen` alone (no header/drawer/footer/children). The screen reads `linking` from `useMeetingLiveSession()` — do not pass it as a prop.
5. Gate chrome: PENDING → `Loadable`; FAILED → `FiAlertCircle` + `semanticColor.stateError`; show org logo **and** name when present; copy under `meetingLayout.linking`.
6. READY shell drawer/header IA belongs to skill `website-meeting-shell` — do not redefine tile sets here.
7. Actions: no-op when `!can.*`; write via internal `batch` only (`STARTED` / `COMPLETED` / ISO timestamps on `me`). Do not export `batch` on the session surface. Do not assign `meeting.*` / `me.*` from UI.
8. Do not re-implement `connect_error` branching here — transport owns it; session only reads `error` / connected / synced.
9. Attend window: mirror `MeetingModel.ATTEND_OPEN_BEFORE_MS` in `meetingAttendWindow.ts`; feed `datetime` + `nowMs` into `resolveMeetingLiveSession`; flip `can.attend` with one `setTimeout` at open time (not a polling interval). `can.enterLive` is navigation-only.
10. Verify with `yarn type-check` in `website/`.
