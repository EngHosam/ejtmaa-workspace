---
name: website-meeting-shell
description: >-
  Ships or extends the organization-host Meeting shell IA: role-based drawer
  tiles (chair vs member/viewer), full-row live tile (Meeting room label),
  in-shell MeetingPage (MeetingPageProvider + Meeting.tsx page switch),
  MeetingInitPage lobby, and header request-to-speak for non-chair. Use when
  editing MeetingDrawerPanel, MeetingHeader, MeetingInitPage under
  meeting/pages, useMeetingPage, useOrganization, Meeting.tsx, or MeetingLayout
  READY chrome. For linking/can/actions/meeting/me use
  website-meeting-live-session; for socket/Yjs use meeting-realtime-socket.
---

# Website Meeting shell

## When to Use

- Changing meeting drawer tiles, labels, icons, or role branching.
- Changing in-shell `MeetingPage` type, provider, or page switch.
- Changing `MeetingInitPage` lobby (org identity, meta, attend).
- Changing org section tint mix (`softLight` in `useOrganization`).
- Changing the header request-to-speak control placement or chrome.
- Wiring a drawer tile from stub (`null`) to a real page under `meeting/pages/`.
- Reviewing Meeting READY shell vs linking gate ownership.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.4–§5.5
- `.cursor/rules/website-meeting-shell.mdc`
- `.cursor/rules/website-meeting-live-session.mdc` (shell mounts only after linking READY)
- `docs/platforms/backend/contracts/meeting-participant-domain.md` §8 (type permissions)
- `.cursor/skills/website-utils-style-prop-audit/SKILL.md` when touching Utils style props

## Instructions

1. Mount header/drawer only under READY shell (`MeetingLayout` / `MeetingShell`). Do not render them on the linking gate.
2. Nest `MeetingPageProvider` in `MeetingLayout` under live/session providers (same chain style). Keep the page switch in `Meeting.tsx`, not in the layout or drawer.
3. Derive tile set from `useMeetingLiveSession().me?.type`: chairperson gets live + talkQueue + attendance + agenda + decisionsAndVote; member and viewer get live + agenda + decisionsAndVote.
4. Never add a participants / role-admin tile. Never add an `init` drawer tile.
5. Keep `live` full-row (`wide` → `gridColumn: 1 / -1`). Drawer label `itemLive` = Meeting room / غرفة الاجتماع (id stays `"live"`).
6. Drawer tiles call `useMeetingPage().setPage(id)` (close mobile overlay via `onClose`). Selected tile: `aria-current="page"` + accent border.
7. New product surfaces: add a page under `components/meeting/pages/` and wire it in `Meeting.tsx`'s switch for that `MeetingPage` id; do not invent a second route.
8. Request-to-speak lives in `MeetingHeader` for MEMBER and VIEWER only (mic + `requestTalk`; accessible name `requestTalkAria`). No switch track. Chairperson uses drawer `talkQueue`. Read `me` from `useMeetingLiveSession()`.
9. Colors from `OrganizationColors` / `defaultOrganizationColors()`. Light section tints: keep `softLight` mix **0.78** in `useOrganization` (fix there, not in page JSX). Style props: `buttonReset` in `baseCssStyle`; behavioral CSS in `cssStyle`.
10. Copy under `ui.layouts.meetingLayout` — `drawer`, `header`, `linking`, and `init` (ar + en, identical keys).
11. `MeetingInitPage`: org logo/name + live `subject`/type/status; attend via `can.attend` / `actions.attend`; attended confirmation (not a disabled button); colors only from `OrganizationColors`.
12. Verify with `yarn type-check` in `website/`.
