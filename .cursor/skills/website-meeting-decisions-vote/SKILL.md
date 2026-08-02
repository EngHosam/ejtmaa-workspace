---
name: website-meeting-decisions-vote
description: >-
  Ships or extends the organization-host Meeting decisions and voting surface:
  phase-explicit capabilities castPreStartVote / castDuringVote /
  managePreStartDecision / manageDuringDecision, session actions castVote /
  setDecisionStatus / clearDecisionVotes, useMeetingDecisions rows with
  adoptStatus, MeetingDecisionsAndVotePage sections, MeetingDecisionCard, the
  Init pre-start ballot gate, the drawer decisionsVoting pulse, and the
  PRE_START seed normalization in MeetingLiveDocHelper. Use when adding or
  fixing a ballot rule, changing who may vote or settle, changing tally
  visibility, wiring durable Decision/Vote persistence, or debugging a decision
  whose button state disagrees with the action. For generic session wiring use
  website-meeting-live-session; for drawer/page IA use website-meeting-shell.
---

# Website Meeting decisions and voting

## When to Use

- Adding, removing, or re-gating a ballot action (`castVote`, `setDecisionStatus`, `clearDecisionVotes`).
- Changing who may cast or settle, when a ballot opens, or when counts become visible.
- Editing `useMeetingDecisions`, `MeetingDecisionsAndVotePage`, `MeetingDecisionCard`, the Init pre-start branch, the `decisionsAndVote` drawer pulse, or `buildLiveDecisions` seed status.
- Wiring the live ballot to durable SQL (`Decision.status`, `Vote` rows, minutes), or adding server-side enforcement.
- Debugging: a chair sees Adopt but the write does nothing; a member can vote before check-in (or cannot); two in-meeting ballots look open at once.

## Do Not Use For

- Session provider / linking mechanics — `website-meeting-live-session`.
- Drawer tile IA and page chrome tokens — `website-meeting-shell`.
- Socket / Yjs transport and the BLOB codec — `meeting-realtime-socket`, `.cursor/rules/meeting-live-state.mdc`.
- Prepare-window decision CRUD (customer portal `createDecision` / `updateDecision`) — that is `flow-customer-meetings.md` §6.3.

## Read first

- `docs/platforms/website/organization-host-routing.md` §5.3 (`can` / `actions` / helpers), §5.5 (page, rows, card, Init branch), §8 limits 2 / 3 / 8
- `.cursor/rules/meeting-decisions-vote.mdc`
- `docs/platforms/backend/contracts/meeting-live-state.md` §1.2 (seed + normalization + shipped writers)
- `docs/platforms/backend/contracts/decision-domain.md`, `vote-domain.md` (why no SQL row is written today)

## Instructions

1. **Confirm the state surface before writing code.** The feature is `decisions[*].status` / `votingType` / nested `votes[memberId]`. No root pointer, no derived list, no mirrored tally, no second store. If the change needs durable data, that is a `Decision` / `Vote` decision — escalate rather than inventing a live field.
2. **Add the capability first, phase-explicit.** Voter gates and chair gates come in pairs (`*PreStart*` / `*During*`); pre-start = meeting live, during = `enterLive` + `STARTED`. Any map-derived input (like `preStartVotesComplete`) is computed once in the session instance and passed into `resolveCan` — never scan `decisions` inside `resolveCan`, and never gate UI on a raw `me?.type`.
3. **Route every gate through `decisionPhaseCan(can, phase)`.** Actions and `useMeetingDecisions` both call it, so the control the user sees and the write the action performs come from the same boolean. Hand-picking a capability key at a call site is how the two drift.
4. **Keep the ballot invariants in the action, not the screen.** Open = `DURING` + `NEW` + `votingType` + no other open `DURING` row. Cast = open + no existing key for me (a cast is final). Settle = open + strict majority for `ACCEPTED` / `REJECTED`, `CANCELED` always. Revote empties `votes` and keeps the ballot open. A screen that re-checks these is a second source of truth.
5. **Derive the outcome once.** `adoptStatus` (`"ACCEPTED" | "REJECTED" | null`) on the row picks the adopt label and icon, disables the control on a tie, and is the argument passed to `setDecisionStatus`. Do not recompute `yes > no` in the card or the page.
6. **Respect tally visibility.** Settled and `PRE_START` rows always show counts; an open `DURING` row shows them on `votingType === "LIVE"`, otherwise chair only. When you add a surface (badge, meta line, aria, notification), decide its secret-ballot behavior before shipping it.
7. **Keep the check-in coupling in one place.** Pre-start ballots gate `can.attend` for voters through `preStartVotesComplete`; the Init prompt reads `hasPendingPreStartVotes` from `useMeetingDecisions`. Do not add a third scan of `meeting.decisions` in a page.
8. **Seed normalization belongs to the backend seed.** `buildLiveDecisions` is the only place that may rewrite a mirrored status (`PRE_START` `NEW` → `UNDER_VOTING`). Never normalize on the client, and never re-normalize an existing non-empty BLOB.
9. **Reuse the shell chrome.** Current-vote panel = talk-queue floor panel chrome; card = agenda/queue card language (`presentCardBackground` + accent border when active, `cardBackground` + `subtleDivider` otherwise). Buttons use the project `disabled` + `disabledStyle` pattern — not a hand-rolled `cursor` / `opacity` pair (`.cursor/rules/website-utils-style-prop-precedence.mdc`).
10. **Copy:** page and card copy under `ui.layouts.meetingLayout.decisions`, drawer aria under `…drawer`, Init prompt under `…init` — ar + en with identical key sets. Status labels must match the backend `enums.decisionStatus` wording, not a new synonym.
11. **Say what is not enforced.** Every gate here is client-side; the socket accepts any roster member's write, so vote finality, one-open-ballot, and chair-only settle are UI contracts. Keep the ceiling rows in `organization-host-routing.md` §8 accurate.
12. **Verify:** `yarn type-check` in `website/` (and `backend/` when the seed changed), then the two-browser pass in `organization-host-routing.md` §11 (pre-start prompt blocks Attend, cast is one-way, open one in-meeting ballot, drawer pulse, adopt on majority, revote, cancel).
