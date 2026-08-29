---
name: website-meeting-shell
description: >-
  Ships or extends the organization-host Meeting shell IA: role-based drawer
  tiles (chair vs member/viewer), full-row live tile (Meeting room label),
  Meeting info → init after live (`HomeMark` monochrome on fill), selected tile soft-accent chrome,
  in-shell MeetingPage (MeetingPageProvider + Meeting.tsx STARTED broadcast stack),
  MeetingInitPage lobby, MeetingLivePage waiting/start only, frosted MeetingPageOverlay,
  MeetingAttendancePage chair attendance log, MeetingAgendaPage live agenda + chair status,
  MeetingTalkQueuePage chair floor + FIFO queue admin,
  MeetingDecisionsAndVotePage pre-start + in-meeting ballots, MeetingPrimaryButton, MeetingMetaChip,
  and the MeetingHeaderMe identity cluster (the header carries no talk control).
  Use when editing MeetingDrawerPanel, MeetingHeader,
  MeetingHeaderMe, MeetingInitPage / MeetingLivePage / MeetingAttendancePage /
  MeetingAgendaPage / MeetingAgendaCard / MeetingTalkQueuePage / MeetingTalkQueueCard /
  MeetingDecisionsAndVotePage / MeetingDecisionCard / MeetingPageStub under meeting/pages,
  MeetingLiveBroadcast, MeetingPageOverlay, MeetingPrimaryButton, MeetingMetaChip,
  useMeetingPage, useMeetingAgenda, useMeetingTalkQueue, useMeetingDecisions, useOrganization,
  HomeMark color overrides for meeting wells,
  Meeting.tsx, or MeetingLayout READY chrome. For linking/can/actions/meeting/me use
  website-meeting-live-session; for talk-queue ordering rules use website-meeting-talk-queue;
  for socket/Yjs use meeting-realtime-socket.
  Init remaining-duration uses session `attendWindow`.
---

# Website Meeting shell

## When to Use

- Changing meeting drawer tiles, labels, icons, Meeting info control, or role branching.
- Changing selected drawer-tile chrome (soft accent fill / partial start rail).
- Changing MeetingHeader chrome or the MeetingHeaderMe identity cluster.
- Shipping or changing Meeting room (`MeetingLivePage` waiting/start, `MeetingLiveBroadcast`, `MeetingPageOverlay`).
- Shipping or changing attendance (`MeetingAttendancePage`, `MeetingAttendanceCard`, `useMeetingAttendance`, shared `FilterCountChips`).
- Shipping or changing agenda (`MeetingAgendaPage`, `MeetingAgendaCard`, `useMeetingAgenda`).
- Shipping or changing the talk-queue page chrome (`MeetingTalkQueuePage`, `MeetingTalkQueueCard`, `useMeetingTalkQueue`).
- Shipping or changing the decisions page chrome (`MeetingDecisionsAndVotePage`, `MeetingDecisionCard`, `useMeetingDecisions`).
- Editing `Meeting.tsx` page switch / STARTED stack, `useMeetingPage`, or MeetingLayout READY chrome.
- Changing org color tokens in `useOrganization` used by meeting chrome.

## Do Not Use For

- Linking / can / actions / meeting / me / attendWindow — `website-meeting-live-session`.
- Socket / Yjs live sync — `meeting-realtime-socket`.

## Instructions

Follow `.cursor/rules/website-meeting-shell.mdc` and `docs/platforms/website/organization-host-routing.md` §5.4–§5.5.

1. Mount header/drawer only under READY shell (`MeetingLayout` / `MeetingShell`). Do not render them on the linking gate. Wrap READY children in `FlexContainer` (same width contract as header/footer `Container`); page bodies use `pv` only — floating overlay adds matching `ph` (`page.padY`). READY shell is viewport-locked (desktop `100vh` / mobile `100dvh`, `overflow: hidden`); exclusive pages and the broadcast stage own internal `customScroll`.
2. Nest `MeetingPageProvider` in `MeetingLayout` under live/session providers (same chain style). Keep the page switch in `Meeting.tsx`, not in the layout or drawer.
3. Derive tile set from `useMeetingLiveSession().me?.type`: chairperson gets live + talkQueue + attendance + agenda + decisionsAndVote; member and viewer get live + agenda + decisionsAndVote.
4. Never add a participants / role-admin tile. Meeting info (`itemHome` + `HomeMark` → `setPage("init")`) is full-row **after** `live` inside the grid — not above it. Label: Meeting info / معلومات الاجتماع.
5. Keep `live` full-row (`wide` → `gridColumn: 1 / -1`). Drawer label `itemLive` = Meeting room / غرفة الاجتماع (id stays `"live"`).
6. Put `disabled` / `livePulse` / `ariaLabel` on tile **defs** before the list (one def reused by both role lists when the tile is shared). Drawer translator only (`meetingLayout.drawer`) — keys `liveLockedAria`, `liveBroadcastingAria`, `agendaDiscussingAria`, `talkQueueActiveAria`, `decisionsVotingAria`. Do not import `init` / `agenda` / `talkQueue` / `decisions` translators into the drawer.
7. Drawer tiles call `useMeetingPage().setPage(id)` (close mobile overlay via `onClose`). Selected tile: soft `sectionAccentBackground` + partial-height start accent rail (`3px`, inset) + `textAccent` label; icon well stays `primaryActionBackground` + white `actionIconOnFill` (no selected accent swap, no full perimeter brand border) + `aria-current="page"`. Meeting info uses the same chrome (`itemHome` + `HomeMark` with `frameClr`/`accentClr` = `actionIconOnFill` — never `FiHome`, never leave the orange floor rail on the fill well).
8. New product surfaces: replace the title stub in the matching `meeting/pages/Meeting*Page.tsx` (keep the file); do not invent a second route. Shared chrome-only placeholder: `MeetingPageStub` (no drawer id mounts it today). `"attendance"`, `"live"`, `"agenda"`, `"talkQueue"`, and `"decisionsAndVote"` are already product UI — extend those pages, do not re-stub them.
9. Header identity: `MeetingHeaderMe` when `me` exists (avatar + name + type chip from session `me`; org colors; type keys). Header-only — never import into attendance cards. Attendance cards own parallel chrome in `MeetingAttendanceCard` (same tokens/language, separate component). Do not import `customer/IdentityAvatar`. Cluster `maxW={14}`. The header has **no** talk control — non-chair raise-hand lives on the broadcast action row and the chair uses the `talkQueue` page. Read `me` from `useMeetingLiveSession()`.
10. Colors from `OrganizationColors` / `defaultOrganizationColors()`. Need a new light/dark fill? Add a token in `useOrganization` — never `colorScheme` branches or opacity hacks in the component. After resolving each seed (or navy/orange fallback), call `softenMeetingPrimary` / `softenMeetingSecondary` from `website/src/app/helpers/ColorHelpers.ts` (hue kept; sat/light pinned to `#2F5D50` / `#8FA99A`) — hook only; do not persist the result. Overlay frost: `pageOverlayBackground` (fixed rgba 0.9) + blur 4px on `MeetingPageOverlay`; floating exclusive pages `bg="@transparent"`. Overlay `z` above the broadcast floating bar. Attendance cards: present → `presentCardBackground`, idle → `idleCardBackground`, type chip → `sectionAccentBackground`. Solid primary wells → `actionIconOnFill` (fixed white). Accent check discs → `accentActionText`. Light section chips: `softLight` **0.78**. Style props: when `bg` is always set (incl. `"@transparent"`), `baseCssStyle={ElementStyles.buttonReset}` only — no redundant `background: "transparent"`; `fontVariantNumeric` / overflow in `cssStyle`.
11. Copy under `ui.layouts.meetingLayout` — `drawer`, `header`, `linking`, `init`, `live`, `broadcast`, `overlay`, `attendance`, `agenda`, `talkQueue`, and `decisions` (ar + en identical keys). Page uses **one** translator namespace for its body. Init = first person; attendance body = third person. Enum status/type strings must match backend `general.ts` labels.
12. `MeetingInitPage` / `MeetingLivePage` (and any exclusive page under the locked shell): root scroll owner uses `ai_c` + `customScroll` — never Flex `center` on that owner (top clip on overflow).
13. `MeetingInitPage`: org logo/name + live meta; when `can.attend` show `MeetingPrimaryButton` → `actions.attend` **and** caption `attendRequiresForRoom` under the button; remaining-duration from session `attendWindow` (never call `useMeetingAttendWindow` here); first-person attended title + relative duration; `roomUnlockedHint` = `textTertiary`. When `can.endMeeting`, colocated `InitEndMeetingSection` → `await confirm(..., "danger")` → `actions.endMeeting` (no broadcast end CTA).
14. Drawer `live`: disable when `!can.enterLive`; `MeetingLivePage` bounce to `"init"` when locked. `MeetingLivePage` is waiting + start only. While `STARTED`, `Meeting.tsx` mounts persistent `MeetingLiveBroadcast` and floats other pages in frosted `MeetingPageOverlay` (full content-column width; scroll `ph={page.padY}` matching page `pv`; dismiss → `"live"`). Do not mount broadcast inside `MeetingLivePage`. When `STARTED` and enterLive, drawer `live` tile shows corner broadcast ping. When any agenda item is `DISCUSSING`, agenda tile pulses.
15. Org primary CTAs in the meeting shell use `MeetingPrimaryButton` (no inline Init/Live button chrome; no `FormActionButton` for org fills). Type/status chips use `MeetingMetaChip` (singular, under `meeting/` — not customer `MeetingMetaChips`). Page-only Init attend/end UI is colocated `InitAttendSection` / `InitEndMeetingSection` in `MeetingInitPage.tsx` (not a `render*` helper, not a shared file). Prefer named colocated components over nested ternaries for multi-branch page sections.
16. `MeetingAttendancePage`: UI only + chair bounce (`setPage("init")` when non-chair). Data via `useMeetingAttendance` (pure helpers colocated; no navigation). Shared `FilterCountChips` (labels from caller; no meeting translator inside chips) + `MeetingAttendanceCard` (not `MeetingHeaderMe`). Utils `Grid` + `SW.max_*`.
17. `MeetingAgendaPage`: `useMeetingAgenda` + `MeetingAgendaCard`. Discussing strip when `DISCUSSING` rows exist; `otherSection` heading only when discussing strip shows and other rows remain. Chair status chips only when `can.setAgendaItemStatus` → `actions.setAgendaItemStatus` (live map only). Subject has no `maxLi`. Page translator `meetingLayout.agenda` only.
18. `MeetingTalkQueuePage`: chair bounce (`setPage("init")` when non-chair) + UI only. Data via `useMeetingTalkQueue`; writes via `actions.giveTalkFloor` / `removeFromTalkQueue` / `endTalkFloor`. Give floor renders on the head row only; Remove renders per row. Page translator `meetingLayout.talkQueue` (type labels from `header.type*`). Ordering, renumbering, and no-auto-promote rules: skill `website-meeting-talk-queue` and `.cursor/rules/meeting-talk-queue.mdc`.
19. `MeetingDecisionsAndVotePage`: all roles, UI only. Data via `useMeetingDecisions` (rows carry their own gates); writes via `actions.castVote` / `setDecisionStatus` / `clearDecisionVotes`. Sections: Current vote panel (talk-queue floor chrome) → in-meeting → pre-start, each with its own `Empty`. Card `MeetingDecisionCard`. Page translator `meetingLayout.decisions` only. Drawer `decisionsAndVote` pulses while a `DURING` ballot is open. Ballot rules: skill `website-meeting-decisions-vote` and `.cursor/rules/meeting-decisions-vote.mdc`.
20. Verify with `yarn type-check` in `website/`.
