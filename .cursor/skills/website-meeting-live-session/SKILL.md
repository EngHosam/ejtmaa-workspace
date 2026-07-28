---
name: website-meeting-live-session
description: >-
  Ships or extends the website MeetingLiveSession surface (stages/can/actions),
  linking PENDING|READY|FAILED gate, and MeetingLinkingScreen under MeetingLayout.
  Use when adding Meeting product UI that must respect linking, wiring
  start/end/attend/left actions, or fixing the org-host meeting gate screen.
  For socket/Yjs transport, use skill meeting-realtime-socket instead.
---

# Website Meeting live session

## When to Use

- Adding Meeting page/shell UI that depends on connection readiness or session capabilities.
- Changing `resolveMeetingLiveSession`, `useMeetingLiveSession`, `MeetingLinkingScreen`, or the linking branch in `MeetingLayout`.
- Wiring start/end/attend/left without screens touching the CRDT directly.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.3 (surface), §5.1 (transport inputs)
- `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/rules/meeting-realtime-socket.mdc` (how `error` / connected / synced are set)
- `.cursor/skills/meeting-realtime-socket/SKILL.md` (socket + Yjs only)

## Instructions

1. Mount consumers under `MeetingLiveProvider` (layout-owned). Do not open a second Meeting socket.
2. Prefer `useMeetingLiveSession()` for `stages` / `can` / `actions`. Use `useMeetingLive` / `useMeetingLiveMe` only for proxy fields or `batch` outside the four actions.
3. Keep capability and stage math in `meetingLiveSession.ts` (`MeetingLiveSessionState`). Hook adds `actions` only.
4. Linking gate: if `stages.linking !== "READY"`, render `MeetingLinkingScreen` alone (no header/drawer/footer/children).
5. Gate chrome: PENDING → `Loadable`; FAILED → `FiAlertCircle` + `semanticColor.stateError`; show org logo **and** name when present; copy under `meetingLayout.linking`.
6. Actions: no-op when `!can.*`; write via `batch` only (`STARTED` / `COMPLETED` / ISO timestamps on `me`).
7. Do not re-implement `connect_error` branching here — transport owns it; session only reads `error` / connected / synced.
8. Verify with `yarn type-check` in `website/`.
