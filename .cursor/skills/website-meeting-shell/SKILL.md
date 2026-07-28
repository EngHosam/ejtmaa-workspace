---
name: website-meeting-shell
description: >-
  Ships or extends the organization-host Meeting shell IA: role-based drawer
  tiles (chair vs member/viewer), full-row live tile, disabled placeholders,
  and header request-to-speak for non-chair. Use when editing MeetingDrawerPanel,
  MeetingHeader, or MeetingLayout READY chrome. For linking/can/actions/meeting/me
  use website-meeting-live-session; for socket/Yjs use meeting-realtime-socket.
---

# Website Meeting shell

## When to Use

- Changing meeting drawer tiles, labels, icons, or role branching.
- Changing the header request-to-speak control placement or chrome.
- Wiring a drawer tile from `available={false}` to a real route/surface.
- Reviewing Meeting READY shell vs linking gate ownership.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.4
- `.cursor/rules/website-meeting-shell.mdc`
- `.cursor/rules/website-meeting-live-session.mdc` (shell mounts only after linking READY)
- `docs/platforms/backend/contracts/meeting-participant-domain.md` §8 (type permissions)
- `.cursor/skills/website-utils-style-prop-audit/SKILL.md` when touching Utils style props

## Instructions

1. Mount header/drawer only under READY shell (`MeetingLayout` / `MeetingShell`). Do not render them on the linking gate.
2. Derive tile set from `useMeetingLiveSession().me?.type`: chairperson gets live + talkQueue + attendance + agenda + decisionsAndVote; member and viewer get live + agenda + decisionsAndVote.
3. Never add a participants / role-admin tile.
4. Keep `live` full-row (`wide` → `gridColumn: 1 / -1`).
5. New tiles stay `available={false}` until that product surface ships.
6. Request-to-speak lives in `MeetingHeader` for MEMBER and VIEWER only (mic + `requestTalk`; accessible name `requestTalkAria`). No switch track. Chairperson uses drawer `talkQueue`. Read `me` from `useMeetingLiveSession()`.
7. Colors from `OrganizationColors` / `defaultOrganizationColors()`. Style props: `buttonReset` in `baseCssStyle`; behavioral CSS in `cssStyle`.
8. Copy under `ui.layouts.meetingLayout.drawer` and `.header` (ar + en, identical keys).
9. Verify with `yarn type-check` in `website/`.
