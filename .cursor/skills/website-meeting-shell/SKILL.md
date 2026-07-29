---
name: website-meeting-shell
description: >-
  Ships or extends the organization-host Meeting shell IA: role-based drawer
  tiles (chair vs member/viewer), full-row live tile (Meeting room label),
  Home → init (`HomeMark` monochrome on fill), selected tile soft-accent chrome,
  in-shell MeetingPage (MeetingPageProvider + Meeting.tsx page switch),
  MeetingInitPage lobby + drawer page stubs, MeetingAttendancePage chair
  attendance log, MeetingHeaderMe identity cluster,
  and header request-to-speak for non-chair. Use when editing MeetingDrawerPanel,
  MeetingHeader, MeetingHeaderMe, MeetingInitPage / MeetingAttendancePage /
  MeetingPageStub under meeting/pages, useMeetingPage, useOrganization, HomeMark
  color overrides for meeting wells, Meeting.tsx, or MeetingLayout READY chrome. For
  linking/can/actions/meeting/me use website-meeting-live-session; for socket/Yjs
  use meeting-realtime-socket. Init remaining-duration uses session `attendWindow`.
---

# Website Meeting shell

## When to Use

- Changing meeting drawer tiles, labels, icons, Home control, or role branching.
- Changing selected drawer-tile chrome (soft accent fill / partial start rail).
- Changing in-shell `MeetingPage` type, provider, or page switch.
- Changing `MeetingInitPage` lobby (org identity, meta, attend) or drawer page stubs.
- Shipping or changing the chair attendance log (`MeetingAttendancePage`, `useMeetingAttendance`, `MeetingAttendanceCard`).
- Changing org section tint mix (`softLight`) or `actionIconOnFill` in `useOrganization`.
- Forcing `HomeMark` / `DrawerMenuIcon` monochrome on solid action wells (`frameClr` / `accentClr`).
- Changing the header identity cluster (`MeetingHeaderMe`) — header-only; never import into attendance cards — or request-to-speak control placement / chrome.
- Wiring a drawer tile from stub (`MeetingPageStub`) to a real page under `meeting/pages/`.
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
4. Never add a participants / role-admin tile. Never put `init` inside the role tile grid — use the Home control above the grid (`setPage("init")`).
5. Keep `live` full-row (`wide` → `gridColumn: 1 / -1`). Drawer label `itemLive` = Meeting room / غرفة الاجتماع (id stays `"live"`).
6. Drawer tiles call `useMeetingPage().setPage(id)` (close mobile overlay via `onClose`). Selected tile: soft `sectionAccentBackground` + partial-height start accent rail (`3px`, inset) + `textAccent` label; icon well stays `primaryActionBackground` + white `actionIconOnFill` (no selected accent swap, no full perimeter brand border) + `aria-current="page"`. Home is the same chrome, full-row above the grid (`itemHome` + `HomeMark` with `frameClr`/`accentClr` = `actionIconOnFill` — never `FiHome`, never leave the orange floor rail on the fill well).
7. New product surfaces: replace the title stub in the matching `meeting/pages/Meeting*Page.tsx` (keep the file); do not invent a second route. Shared chrome-only placeholder: `MeetingPageStub`. `"attendance"` is already product UI — extend that page, do not re-stub it.
8. Header identity: `MeetingHeaderMe` when `me` exists (avatar + name + type chip from session `me`; org colors; type keys). Header-only — never import into attendance cards. Attendance cards own parallel chrome in `MeetingAttendanceCard` (same tokens/language, separate component). Do not import `customer/IdentityAvatar`. Cluster `maxW={14}`. Request-to-speak lives in `MeetingHeader` for MEMBER and VIEWER only. Read `me` from `useMeetingLiveSession()`.
9. Colors from `OrganizationColors` / `defaultOrganizationColors()`. Need a new light/dark fill? Add a token in `useOrganization` — never `colorScheme` branches or opacity hacks in the component. Attendance cards: present → `presentCardBackground`, idle → `idleCardBackground`, type chip → `sectionAccentBackground`. Solid primary wells → `actionIconOnFill` (fixed white). Accent check discs → `accentActionText`. Light section chips: `softLight` **0.78**. Style props: when `bg` is always set (incl. `"@transparent"`), `baseCssStyle={ElementStyles.buttonReset}` only — no redundant `background: "transparent"`; `fontVariantNumeric` / overflow in `cssStyle`.
10. Copy under `ui.layouts.meetingLayout` — `drawer`, `header`, `linking`, `init`, and `attendance` (incl. `title`; ar + en identical keys). Page uses **one** `attendance` translator for page copy. Init = first person; attendance body = third person.
11. `MeetingInitPage`: org logo/name + live meta; attend via `can.attend` / `actions.attend`; remaining-duration from session `attendWindow` (never call `useMeetingAttendWindow` here); first-person attended title + relative duration; `roomUnlockedHint` = `textTertiary`.
12. Drawer `live`: disable when `!can.enterLive`; `MeetingLivePage` bounce to `"init"` when locked.
13. `MeetingAttendancePage`: UI only + chair bounce (`setPage("init")` when non-chair). Data via `useMeetingAttendance` (pure helpers colocated; no navigation). Shared `FilterCountChips` (labels from caller; no meeting translator inside chips) + `MeetingAttendanceCard` (not `MeetingHeaderMe`). Utils `Grid` + `SW.max_*`.
14. Verify with `yarn type-check` in `website/`.
