# Meeting Live State (CRDT)

## Scope

The collaborative state of a live meeting: the Yjs document that carries `subject`, `type`, `status`, `datetime`, `minMembersCount`, `participants`, `currentTalkMemberId`, `agendaItems`, `currentAgendaItemId`, `decisions`, and `currentDecisionId` while the meeting runs, the `meetings.live_state` BLOB that persists it, and the in-memory registry that owns the document per process.

Transport (namespace, handshake, events, authorization): `docs/platforms/backend/contracts/meeting-realtime-socket.md`.
Website consumer (`MeetingLiveProvider` / `useMeetingLive` / `useMeetingLiveSession`, SyncedStore): `docs/platforms/website/organization-host-routing.md` §5.1, §5.3.
Meeting domain (columns, GQL, requester write path): `docs/platforms/backend/contracts/meeting-domain.md`.

Dependency: `yjs@13.6.27` (`backend/package.json`), same major/minor as the website copy.

## 1) Model surface

File: `backend/src/app/orm/models/Meeting.ts`.

### 1.1 Column

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `live_state` | `BLOB` | yes | `null` | Yjs V2 state snapshot of the whole document; `null` until the first persist |

Declared in the `Attrs` type under a `//live session` comment group, separate from `//info`. The ORM model owns **only** this column — codec, seed, registry live in `MeetingLiveDocHelper` (§1.3, §2); map name and writable statuses live in the mirrored `types/meeting.ts` (§1.2).

### 1.2 Live map contract (mirrored)

File pair: `backend/src/app/types/meeting.ts` ↔ `website/src/types/meeting.ts` — **byte-for-byte identical**, no external imports.

| Export | Value / shape | Role |
|---|---|---|
| `MEETING_LIVE_MAP` | `"meeting"` | Root `Y.Map` name and SyncedStore root key. Backend `doc.getMap(MEETING_LIVE_MAP)`; website `syncedStore({ [MEETING_LIVE_MAP]: {} })` |
| `MEETING_LIVE_STATUSES` | `readonly ["WAITING_TO_START", "STARTED"]` | Write gate for `meeting.live.update` (`meeting-realtime-socket.md` §4) |
| `MeetingLiveWritableStatus` | union of the statuses above | Type companion |
| `MeetingLiveTypes` / `MeetingLiveType` | `PERIODIC` \| `EMERGENCY` | Live `type` field |
| `MeetingLiveStatuses` / `MeetingLiveStatus` | full meeting status union | Live `status` field |
| `MeetingLiveParticipantTypes` / `MeetingLiveParticipantType` | `CHAIRPERSON` \| `MEMBER` \| `VIEWER` | Per-roster `type` |
| `MeetingLiveConnectionStatuses` / `MeetingLiveConnectionStatus` | `ONLINE` \| `OFFLINE` | Per-roster connection |
| `MeetingLiveParticipant` | see below | One roster entry |
| `MeetingLiveAgendaItemStatuses` / `MeetingLiveAgendaItemStatus` | `WAITING` \| `DISCUSSING` \| `DONE` \| `CANCELED` | Per-agenda-item session status |
| `MeetingLiveAgendaItem` | see below | One agenda line in the live map |
| `MeetingLiveDecisionPhases` / `MeetingLiveDecisionPhase` | `PRE_START` \| `DURING` | Decision phase (SQL mirror) |
| `MeetingLiveDecisionStatuses` / `MeetingLiveDecisionStatus` | `NEW` \| `UNDER_VOTING` \| `ACCEPTED` \| `REJECTED` \| `CANCELED` | Decision voting lifecycle (SQL mirror + live `CANCELED`) |
| `MeetingLiveDecisionVotingTypes` / `MeetingLiveDecisionVotingType` | `LIVE` \| `SECRET` | Optional voting type (SQL mirror) |
| `MeetingLiveVoteValues` / `MeetingLiveVoteValue` | `YES` \| `NO` | Ballot value (SQL `voteValue` mirror) |
| `MeetingLiveVote` | see below | One cast under a decision (`votes` key = member id) |
| `MeetingLiveDecision` | see below | One decision line (+ nested empty `votes` at seed) |
| `MeetingLiveMap` | see below | Full live document shape |

```ts
type MeetingLiveParticipant = {
    id: string;
    type: MeetingLiveParticipantType;
    name: string;
    avatarUrl: string | null;
    attendedAt: string | null;   // ISO-8601 or null
    leftAt: string | null;
    connectionStatus: MeetingLiveConnectionStatus;
    onlineAt: string | null;
    offlineAt: string | null;
    talkTurn: number | null;     // null = not in live talk queue; number = queue order
};

type MeetingLiveAgendaItem = {
    id: string;
    sortOrder: number;
    subject: string;
    status: MeetingLiveAgendaItemStatus; // WAITING | DISCUSSING | DONE | CANCELED
    isLiveCreated: boolean;
};

type MeetingLiveVote = {
    memberId: string;
    value: MeetingLiveVoteValue; // YES | NO
    castAt: string;              // ISO-8601
};

type MeetingLiveDecision = {
    id: string;
    sortOrder: number;
    subject: string;
    phase: MeetingLiveDecisionPhase;     // PRE_START | DURING (from SQL)
    status: MeetingLiveDecisionStatus;   // from SQL; live may set CANCELED
    votingType: MeetingLiveDecisionVotingType | null; // from SQL
    isLiveCreated: boolean;              // session-only
    votes: Record<string, MeetingLiveVote>; // keyed by member id; empty at seed
};

type MeetingLiveMap = {
    subject: string;
    type: MeetingLiveType;
    status: MeetingLiveStatus;
    datetime: string;            // ISO-8601 from SQL Meeting.datetime
    minMembersCount: number;     // from SQL Meeting.min_members_count
    participants: Record<string, MeetingLiveParticipant>; // keyed by member id
    currentTalkMemberId: string | null; // null = nobody speaking; else participants key
    agendaItems: Record<string, MeetingLiveAgendaItem>;   // keyed by agenda item id
    currentAgendaItemId: string | null;
    decisions: Record<string, MeetingLiveDecision>;       // keyed by decision id (all phases)
    currentDecisionId: string | null; // mainly DURING chair one-by-one; null = none
};
```

Attendance is timestamp-only (`attendedAt` / `leftAt`); there is no separate boolean. Connection presence fields are seeded `OFFLINE` / null timestamps on first document create; writers that flip online/offline are **not** shipped yet (`meeting-realtime-socket.md` shipped limits).

`talkTurn` is the live talk-queue order on the roster entry (`null` = not queued). Root `currentTalkMemberId` is who is speaking now (`null` = nobody). Both are session-only (not SQL `talk_records` columns); they die on session reset (§3.1). Durable queue/history remains `TalkRecord` (`talk-record-domain.md`). Writers that flip talk-queue fields are **not** shipped yet. Existing non-empty BLOBs are decoded as-is (no talk-field backfill).

`datetime` is the scheduled start used by the website client session gate for the self-check-in open window (`MeetingModel.ATTEND_OPEN_BEFORE_MS` — 30 minutes before start). It is seeded from SQL and is **not** a collaborative edit target.

`minMembersCount` is the quorum denominator for the chair attendance log UI. It is seeded from SQL `min_members_count` on **first empty** `live_state` create only and is **not** a collaborative edit target. Existing non-empty BLOBs are decoded as-is (no scalar backfill); clients must treat a missing value as “hide quorum strip,” not invent a default.

`agendaItems` mirrors durable SQL `AgendaItem` lines (`id` / `sort_order` → `sortOrder` / `subject`) on **first empty** `live_state` create. Per-item `status` defaults to `WAITING`; `isLiveCreated` defaults to `false`. Session cancel of a line uses `status: "CANCELED"` (no live delete). Those flags and statuses, with root `currentAgendaItemId` (seeded `null`), exist **only** in the live map — not on SQL `agenda_items`. They die on session reset (§3.1). Writers that flip agenda status, live-mutation flags, or current id are **not** shipped yet. Existing non-empty BLOBs are decoded as-is (no agenda backfill).

`decisions` mirrors durable SQL `Decision` rows (**both** `PRE_START` and `DURING`) on **first empty** `live_state` create: `sort_order` → `sortOrder`, `subject`, `phase`, `status`, `voting_type` → `votingType`. Session-only `isLiveCreated` defaults `false`. Session cancel of a decision uses live `status: "CANCELED"` (map-only; not a SQL decision status). Per-decision `votes` seeds as an **empty** nested collaborative `Y.Map` (logical `{}`) — **no** per-participant vote slots and **no** SQL `Vote` backfill at seed. A cast appears under `votes[memberId]` only when a writer adds it later (writers **not** shipped). Root `currentDecisionId` seeds `null` (chair one-by-one pointer for `DURING`; `PRE_START` list voting does not require it). Product: `PRE_START` = full list members may vote; `DURING` = chair advances one decision at a time via `currentDecisionId`. Durable casts remain SQL `Vote` (`vote-domain.md`). Existing non-empty BLOBs decode as-is (no decision/votes backfill).

### 1.3 Document codec and seed

File: `backend/src/app/helpers/MeetingLiveDocHelper.ts` (same module as the registry). Imports `MEETING_LIVE_MAP` / live types from the mirror (§1.2).

Codec/seed helpers are **module-private**. Public surface of this file is the registry (§2) plus `readLiveFields` (deferred column apply, §6).

| Member | Role |
|---|---|
| `createLiveDoc(fields)` | Private. One `transact`: nested `participants` then `currentTalkMemberId`; nested `agendaItems` then `currentAgendaItemId`; nested `decisions` (each with nested empty `votes` `Y.Map`) then `currentDecisionId` |
| `encodeLiveDoc` / `decodeLiveDoc` | Private. V2 BLOB codec |
| `buildLiveParticipants(meeting)` | Private. `meeting.getParticipants({ include: [{ association: "member", required: true }] })` → `Record` keyed by `member_id`; seeds `connectionStatus: "OFFLINE"`, `talkTurn: null` |
| `buildLiveAgendaItems(meeting)` | Private. `meeting.getAgendaItems()` → `Record` keyed by agenda `id`; seeds `status: "WAITING"`, `isLiveCreated: false` |
| `buildLiveDecisions(meeting)` | Private. `meeting.getDecisions()` → `Record` keyed by decision `id` (all phases); maps SQL fields; seeds `isLiveCreated: false`, `votes: {}` (no SQL Vote seed; no per-member slots) |
| `getLiveDoc(meeting)` | Private. Non-empty `live_state` → `decodeLiveDoc` (no backfill); else `createLiveDoc` with participants / talk / agenda / decisions seeds + null current-* ids |
| `readLiveFields(doc)` | **Exported.** `doc.getMap(MEETING_LIVE_MAP).toJSON() as Partial<MeetingLiveMap>` — only sanctioned read-back |

Nested `Y.Map` for each participant, agenda item, decision, and per-decision `votes` map is required so collaborative field updates merge by identity. A plain object under those keys would not be a collaborative map. Seed must not store plain `{}` as the Yjs value of `votes` — use an empty `Y.Map` (logical seed input may still be `votes: {}`).

SQL column enums remain `MeetingType` / `MeetingStatus` from `G_Tr` keys; the CRDT document uses `MeetingLiveMap` only.

**Encoding is V2 on every hop** — BLOB, sync response, and update broadcast. A V1 update applied with `applyUpdateV2` (or the reverse) throws; the website converts its V1 doc events with `Y.convertUpdateFormatV1ToV2` before emitting.

`readLiveFields` has **no caller today**. It exists for the deferred column apply (§6).

## 2) Registry and persistence

File: `backend/src/app/helpers/MeetingLiveDocHelper.ts`.

Two module-level maps keyed by `meetingId`: `entries` (live documents) and `inflight` (in-progress loads). `PERSIST_DEBOUNCE_MS = 1500`.

| Export | Behavior |
|---|---|
| `getOrCreateMeetingLiveDoc(meetingId)` | Returns the cached entry; otherwise loads it once. Concurrent callers await the same `inflight` promise, and a load that lost a race returns the entry that won |
| `peekMeetingLiveDoc(meetingId)` | Cached entry or `undefined`; never loads |
| `schedulePersistMeetingLiveDoc(meetingId)` | Restarts the debounce timer |
| `flushPersistMeetingLiveDoc(meetingId)` | Clears the timer and awaits `meeting.update({ live_state: encodeLiveDoc(doc) })` |
| `destroyMeetingLiveDoc(meetingId, { flush })` | Flushes (default) or cancels the timer, then `doc.destroy()` and drops the entry |

Load path inside `getOrCreateMeetingLiveDoc`:

1. `Meeting().findByPk(meetingId)`; a missing row throws `"404"`.
2. Records whether the row already had a non-empty `live_state`.
3. `getLiveDoc(meeting)` (§1.3) and stores `{ doc, meeting }` in `entries`.
4. Subscribes `doc.on("update")` → `schedulePersistMeetingLiveDoc`.
5. Schedules one persist when the row had **no** BLOB, so the seeded document reaches the column even if nobody edits.

The timer callback is not awaited by anyone, so a failed write is caught and reported through `Log().unhandledError(err)` instead of becoming an unhandled rejection.

The Sequelize row captured at load is the same instance the flush writes through. It is a **snapshot**: columns changed elsewhere afterwards are not reflected in it.

## 3) Lifecycle

1. A meeting is created and edited through `MeetingRequester`; SQL columns are the only truth and `live_state` stays `null` (`meeting-domain.md` §9).
2. First `meeting.live.sync` on the meeting: the registry seeds a document from `subject` / `type` / `status` and `buildLiveParticipants()`, then schedules the first BLOB write.
3. While the meeting is live, the document is the source of truth for those map fields. Every accepted update mutates the document, is broadcast to the room, and restarts the debounce.
4. The BLOB trails the document by up to the debounce window (§7.1).
5. Reconnect or a second participant replays the whole state through the sync handshake; nothing is read from the SQL columns again while the entry stays in memory.
6. Applying the live fields back onto the SQL columns is **not shipped** (§6).

### 3.1 Requester-driven reset

`MeetingRequester` resets the document at the two lifecycle points where the preparation state changes underneath it (`meeting-domain.md` §9.1a):

| Trigger | SQL in the transaction | After commit |
|---|---|---|
| `approve` (`DRAFT` → `WAITING_TO_START`) | `live_state = null` | `destroyMeetingLiveDoc(meetingId, { flush: false })` |
| `demoteApprovedMeetingToDraft` (any content write on a `WAITING_TO_START` meeting) | `status = DRAFT`, `live_state = null` | same call |

Two properties are load-bearing:

- **`flush: false`** — the default (`true`) would encode the in-memory document back into the BLOB the transaction just cleared, resurrecting the state the reset removed.
- **`transaction.afterCommit`** — the registry is process memory and cannot be rolled back, so eviction must not run for a transaction that later fails.

A `DRAFT` meeting is outside `MEETING_LIVE_STATUSES`, so no session can be open at approve time; the call exists to evict an entry left behind by an earlier approve → demote → approve cycle.

## 4) GraphQL exposure

`live_state` is deliberately **not** part of any GraphQL contract.

```ts
// backend/src/app/gql/bridges/customer/MeetingBridge.ts
static registerOrmAttrs = {
    expect: ["live_state"]
};
```

`BridgeBase.bootRegisteredAttrs()` auto-registers every ORM attribute as a bridge attr; `registerOrmAttrs.expect` is the **exclusion** list (`expect.indexOf(k) >= 0 → continue`). Without the entry the raw CRDT buffer would be auto-mapped onto the bridge, so any ORM column that must never leave through GraphQL has to be listed here.

`_Meeting` SDL is unchanged: no `live_state` field, no new codegen output.

## 5) Failure modes

| Scenario | Behavior |
|---|---|
| Meeting row deleted while a session is open | `getOrCreateMeetingLiveDoc` throws `"404"` on the next sync/update; the client sees the framework socket error, not `meeting.live.error` |
| BLOB unreadable or truncated | `decodeLiveDoc` → `Y.applyUpdateV2` throws inside `getLiveDoc()`; the load rejects and the entry is not cached |
| DB write fails in the debounce timer | Logged via `Log().unhandledError`; the document stays live in memory and the next update reschedules the write |
| Two concurrent first loads of the same meeting | `inflight` dedupes; a racing loser reuses the cached entry instead of creating a second document |
| Process restart | Everything since the last flush is lost; the next load replays the BLOB (§7.2) |

## 6) Deferred: applying live fields to SQL columns

Product decision: meeting-row fields `subject`, `type`, and `status` are **not** written back from the live document while the session runs. The reflection happens once when the meeting is completed, and that step is not implemented yet.

What it will own when it lands:

- read the live values through `readLiveFields(doc)`,
- write the meeting columns and the terminal `status` atomically with whatever else completion touches,
- flush and evict the registry entry (`destroyMeetingLiveDoc`) so no writer keeps mutating a completed meeting,
- close the stale-status window described in §7.3.

`participants` in the live map is a collaborative roster mirror (display + connection/attendance timestamps). SQL attendance on `meeting_participants` remains a separate write path; this deferred step is not a blanked “apply entire `MeetingLiveMap` to Meeting columns.”

Until then the SQL meeting columns keep the values the requester write path left, and consumers that read a meeting through GraphQL see those, not the live edits.

## 7) Shipped limits (intentional)

1. **Debounce has no maximum wait.** Every document update restarts the 1500 ms timer, so a continuously edited meeting keeps deferring its BLOB write until the editing pauses.
2. **No flush on shutdown.** Nothing hooks process exit; unflushed changes are lost on restart.
3. **Stale status gate.** The update gate reads the meeting row captured at handshake time (`socket.data.meeting`), so a status change made outside the live session is not seen until the socket reconnects. Acceptable because completion runs through the live session itself (§6).
4. **Eviction only on approve / demotion.** `destroyMeetingLiveDoc` is called from those two requester paths (§3.1); `peekMeetingLiveDoc` still has no caller. Nothing evicts an entry when a session simply ends, so memory still grows with the number of meetings touched until the process restarts.
5. **Single instance.** The registry is process memory. Two backend processes would each hold an independent document for the same meeting and overwrite each other's BLOB; horizontal scaling needs a shared document plane first.
6. **Seed write on first load.** A meeting whose BLOB is `null` gets one write on first sync even when nobody edits. Harmless because a session only opens for a live meeting, whose columns are no longer edited through the form.
7. **No automatic BLOB schema migration.** A non-empty `live_state` is decoded as-is. Documents created before `participants` / `datetime` / `minMembersCount` / `agendaItems` / `currentAgendaItemId` / `talkTurn` / `currentTalkMemberId` / `decisions` / `currentDecisionId` / nested `votes` existed are not back-filled on load; a requester reset (§3.1) clears the BLOB so the next sync re-seeds from SQL.

## 8) Traceability

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Meeting.ts` | `live_state` column only | §1.1 |
| `backend/src/app/types/meeting.ts` | mirrored live map contract (`MeetingLiveMap`, participants, `MEETING_LIVE_*`) | §1.2; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | private codec+seed + registry + exported `readLiveFields` | §1.3, §2 |
| `backend/src/app/orm/models/Customer.ts` | Ability notify-template roster `include: ["member"]` | §1.3; `meeting-domain.md` §9.1 |
| `backend/src/app/orchestrator/requesters/MeetingRequester.ts` | `live_state = null` + `afterCommit` destroy on approve and on demotion to draft | §3.1 |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | `registerOrmAttrs.expect: ["live_state"]` | §4 |
| `backend/eng-hosam/@nodejs/gql/src/BridgeBase.ts` | `bootRegisteredAttrs()` exclusion semantics (library, not edited) | §4 |
| `backend/package.json` | `yjs` dependency pin | Scope |
| `backend/src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | reads the registry document | `meeting-realtime-socket.md` §3 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | writes the registry document, enforces `MEETING_LIVE_STATUSES` | `meeting-realtime-socket.md` §3–§4 |

## 9) Change set inventory (this delivery)

Working-tree delivery that relocated live codec out of `Meeting`, extended `MeetingLiveMap` with `participants`, and aligned website SyncedStore to `MEETING_LIVE_MAP`. Earlier meeting-realtime delivery paths remain described in §§1–8 and in `meeting-realtime-socket.md` / `organization-host-routing.md`.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/types/meeting.ts` | modified — `MEETING_LIVE_MAP` / `MEETING_LIVE_STATUSES` / `MeetingLiveParticipant*` / `participants` / `datetime` on `MeetingLiveMap` | §1.2 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — private codec+seed (incl. nested participants + `datetime`), registry unchanged in role | §1.3, §2 |
| `src/app/orm/models/Meeting.ts` | modified — removed live-session statics/instance helpers; keeps `live_state` column; schedule + `ATTEND_OPEN_BEFORE_MS` statics live on the model | §1.1; `meeting-domain.md` §3.2b |
| `src/app/orm/models/Customer.ts` | modified — Ability notify-template path `include: ["member"]` (association name, not model class) | §1.3; `meeting-domain.md` Ability notify gate |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | modified — write gate uses `MEETING_LIVE_STATUSES` from mirror | `meeting-realtime-socket.md` §3.2, §4 |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/types/meeting.ts` | modified — identical mirror to backend | §1.2; `organization-host-routing.md` §5.1 |
| `src/app/ui/components/meeting/hooks/useMeetingLive.tsx` | modified — SyncedStore shape `{ [MEETING_LIVE_MAP]: Partial<MeetingLiveMap> }`; root init `{}`; read `liveStore[MEETING_LIVE_MAP]` | `organization-host-routing.md` §5.1 |
| `lib/tsconfig.tsbuildinfo` | modified — incremental cache from `yarn type-check`; not behavioral | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-live-state.md` | this page | — |
| `docs/platforms/backend/contracts/meeting-realtime-socket.md` | map fields + status gate naming | §Related |
| `docs/platforms/backend/contracts/meeting-domain.md` | live column ownership + helper pointer | §Related |
| `docs/platforms/website/organization-host-routing.md` | SyncedStore / mirror consumer | §5.1 |
| `.cursor/rules/meeting-live-map-mirror.mdc` | modified — mirror pair includes map consts + participants | governance |
| `.cursor/rules/meeting-live-state.mdc` | modified — codec ownership, SyncedStore empty Partial root, nested participant Y.Maps | governance |
| `.cursor/rules/meeting-realtime-socket.mdc` | modified — status gate from types; SyncedStore key contract | governance |
| `.cursor/rules/sequelize-include-by-association-name.mdc` | **added** — include by association name | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | modified — live map / SyncedStore / helper ownership instructions | governance |

Untracked build output under `backend/lib/`, `backend/.exporters/`, `backend/.types/`, `backend/.webpack_root.ts` and `website/server/` is generated by build scripts and is not part of the source contract.

## 9b) Change set inventory (live `datetime` seed)

| Path | State | Where described |
|---|---|---|
| `backend/src/app/types/meeting.ts` | modified — `MeetingLiveMap.datetime` | §1.2 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `datetime` on first create; decode existing BLOB as-is (no scalar backfill) | §1.3, §7 |
| `backend/src/app/orm/models/Meeting.ts` | modified — `ATTEND_OPEN_BEFORE_MS` | `meeting-domain.md` §3.2b |
| `website/src/types/meeting.ts` | modified — identical mirror | §1.2 |
| `docs/platforms/backend/contracts/meeting-live-state.md` | this page | — |
| `docs/platforms/backend/contracts/meeting-participant-domain.md` | self-check-in open window | §3.6 there |
| `docs/platforms/website/organization-host-routing.md` | client `can.attend` / room gate | §5.3–§5.5, §10d |

## 9c) Change set inventory (live `minMembersCount` seed)

| Path | State | Where described |
|---|---|---|
| `backend/src/app/types/meeting.ts` | modified — `MeetingLiveMap.minMembersCount` | §1.2 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | modified — seed `minMembersCount` on first create; decode existing BLOB as-is (no scalar backfill) | §1.3, §7 |
| `website/src/types/meeting.ts` | modified — identical mirror | §1.2 |
| `docs/platforms/backend/contracts/meeting-live-state.md` | this page | — |
| `docs/platforms/website/organization-host-routing.md` | chair attendance quorum strip | §5.5, §10f |

## 9d) Change set inventory (live agenda map — shipped)

**Shipped behavior (final).** On first empty `live_state` create, `MeetingLiveDocHelper` seeds nested `agendaItems` from SQL `AgendaItem` rows (`id`, `sort_order` → `sortOrder`, `subject`) with session defaults `status: "WAITING"`, `isLiveCreated: false`, and root `currentAgendaItemId: null`. Nested per-id `Y.Map`s (same pattern as `participants`). `createLiveDoc` sets `currentAgendaItemId` **after** `agendaItems`. Session cancel of a line is `status: "CANCELED"` — **no** live delete / **no** `isDeleted`. `isLiveCreated` means the chair created the line **during the live session** (name chosen so it is not confused with SQL authoring). Existing non-empty BLOBs decode as-is (no agenda backfill). Session reset (§3.1) clears the BLOB so the next sync re-seeds. **Not shipped:** client/server writers that flip agenda `status`, live-mutation flags, or `currentAgendaItemId`; agenda UI on `MeetingAgendaPage` (still stub). SQL `agenda_items` columns unchanged.

### Exhaustive path inventory

| Path | Repo | State | Where described |
|---|---|---|---|
| `backend/src/app/types/meeting.ts` | backend | modified — `MeetingLiveAgendaItemStatuses` / `MeetingLiveAgendaItem` / `agendaItems` / `currentAgendaItemId` | §1.2 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | backend | modified — `buildLiveAgendaItems`; nested seed; `currentAgendaItemId` after `agendaItems` | §1.3, §7 |
| `website/src/types/meeting.ts` | website | modified — byte-identical mirror | §1.2; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `website/lib/tsconfig.tsbuildinfo` | website | modified — TypeScript incremental build cache only | **excluded from narrative** (generated noise from `yarn type-check`) |
| `docs/platforms/backend/contracts/meeting-live-state.md` | root | modified — this page | — |
| `docs/platforms/backend/contracts/agenda-item-domain.md` | root | modified — SQL vs live-map session fields; cancel via `CANCELED` | scope + §2 + related |
| `docs/platforms/website/organization-host-routing.md` | root | modified — SyncedStore / shipped live-map field list; §10n | §5.1, shipped limits, §10n |
| `.cursor/rules/meeting-live-state.mdc` | root | modified — nested `agendaItems`; map-only session fields; no live delete | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | root | modified — mirror includes agenda types | governance |
| `.cursor/rules/agenda-item-meeting-child.mdc` | root | modified — live session fields vs SQL | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | root | modified — seed/mirror checklist for agenda | skill |

### Triage notes

| Symptom | Likely cause |
|---|---|
| Client `meeting.agendaItems` missing after sync | Non-empty pre-agenda `live_state` BLOB (no backfill). Reset via approve/demote (§3.1) or clear BLOB so next sync re-seeds |
| Flags / `currentAgendaItemId` never change | Expected — writers not shipped; seed only |
| Expecting SQL column for agenda discussion status | Wrong plane — `status` / `isLiveCreated` / `currentAgendaItemId` are live-map only |

## 9e) Change set inventory (live talk queue fields — shipped)

**Shipped behavior.** On first empty `live_state` create, each live participant seeds `talkTurn: null` (not in queue). Root `currentTalkMemberId` seeds `null` (nobody speaking) and is set in `createLiveDoc` **after** `participants`. Session-only; not SQL `talk_records`. Durable talk history remains `TalkRecord`. **Not shipped:** writers / talk-queue UI.

| Path | Repo | State | Where described |
|---|---|---|---|
| `backend/src/app/types/meeting.ts` | backend | modified — `talkTurn` / `currentTalkMemberId` | §1.2 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | backend | modified — seed `talkTurn: null`; `currentTalkMemberId` after `participants` | §1.3 |
| `website/src/types/meeting.ts` | website | modified — identical mirror | §1.2 |
| `website/lib/tsconfig.tsbuildinfo` | website | may change from type-check | **excluded** (generated) |
| `docs/platforms/backend/contracts/meeting-live-state.md` | root | this page | — |
| `docs/platforms/backend/contracts/talk-record-domain.md` | root | SQL vs live talk fields | scope + Related |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | root | plane table notes live talk fields | planes |
| `docs/platforms/website/organization-host-routing.md` | root | live map field list; §10o | §5.1 / §8 / §10o |
| `docs/platforms/website/README.md` | root | change-set pointer → §10o | website index |
| `docs/platforms/backend/README.md` | root | index blurbs | backend index |
| `docs/README.md` | root | live-state index blurb | root index |
| `.cursor/rules/meeting-live-state.mdc` | root | talk session fields | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | root | mirror includes talk fields | governance |
| `.cursor/rules/talk-record-meeting-child.mdc` | root | live vs SQL | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | root | seed order checklist (6d) | skill |

### Triage notes (talk queue)

| Symptom | Likely cause |
|---|---|
| Client missing `talkTurn` / `currentTalkMemberId` after sync | Non-empty pre-talk `live_state` BLOB (no backfill). Reset via approve/demote (§3.1) |
| `talkTurn` / `currentTalkMemberId` never change | Expected — writers not shipped; seed only |
| Expecting SQL column for live queue position / current speaker | Wrong plane — session fields are live-map only; durable history is `TalkRecord` |

## 9f) Change set inventory (live decisions + empty votes map — shipped)

**Shipped behavior.** On first empty `live_state` create, seed nested `decisions` from **all** SQL `Decision` rows (`PRE_START` + `DURING`) with SQL `phase` / `status` / `voting_type` → `votingType`, plus `isLiveCreated: false`. Live status union includes `CANCELED` (session cancel; not SQL). Per-decision `votes` seeds as an **empty** nested `Y.Map` (logical `votes: {}`) — no member slots, no SQL `Vote` seed. Root `currentDecisionId: null` set **after** `decisions`. Product: `PRE_START` = full list voting; `DURING` = chair one-by-one via `currentDecisionId`. **Not shipped:** decision/vote writers; `decisionsAndVote` UI (stub). Durable casts remain SQL `Vote`.

| Path | Repo | State | Where described |
|---|---|---|---|
| `backend/src/app/types/meeting.ts` | backend | modified — `MeetingLiveDecision*` / `MeetingLiveVote*` / `decisions` / `currentDecisionId` | §1.2 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | backend | modified — `buildLiveDecisions`; nested empty `votes` `Y.Map`; seed order | §1.3 |
| `website/src/types/meeting.ts` | website | modified — identical mirror | §1.2 |
| `website/lib/tsconfig.tsbuildinfo` | website | may change from type-check | **excluded** |
| `docs/platforms/backend/contracts/meeting-live-state.md` | root | this page | — |
| `docs/platforms/backend/contracts/decision-domain.md` | root | SQL vs live | scope |
| `docs/platforms/backend/contracts/vote-domain.md` | root | durable vs live votes nest | scope |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | root | plane table notes | planes |
| `docs/platforms/website/organization-host-routing.md` | root | §10p | §5.1 / §8 / §10p |
| `docs/platforms/website/README.md` | root | pointer → §10p | website index |
| `docs/platforms/backend/README.md` | root | index blurb | backend index |
| `docs/README.md` | root | live-state blurb | root index |
| `.cursor/rules/meeting-live-state.mdc` | root | decisions + empty votes nest | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | root | mirror includes decisions/votes | governance |
| `.cursor/rules/decision-meeting-child.mdc` | root | live vs SQL | governance |
| `.cursor/rules/vote-decision-child.mdc` | root | live nest note | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | root | 6e decisions/votes seed | skill |

### Triage notes (decisions / votes)

| Symptom | Likely cause |
|---|---|
| Client missing `decisions` / `currentDecisionId` / nested `votes` | Pre-decision `live_state` BLOB (no backfill). Reset via approve/demote (§3.1) |
| `votes` always empty | Expected at seed — writers not shipped; no SQL Vote backfill |
| Expecting per-member empty vote slots at seed | Rejected — cast key appears only when a writer adds it |

## 10) Related

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-domain.md` — columns, GQL read, requester write path, `ATTEND_OPEN_BEFORE_MS`
- `docs/platforms/backend/contracts/agenda-item-domain.md` — durable SQL agenda; live map session fields
- `docs/platforms/backend/contracts/decision-domain.md` — durable SQL decisions; live `decisions` / `currentDecisionId` / nested `votes`
- `docs/platforms/backend/contracts/vote-domain.md` — durable SQL casts; live nest under `decisions[*].votes` (empty at seed)
- `docs/platforms/backend/contracts/talk-record-domain.md` — durable SQL talk queue/history; live `talkTurn` / `currentTalkMemberId`
- `docs/platforms/backend/contracts/livekit-media-plane.md` — LiveKit is not agenda/talk/decision/vote truth
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.3, §10n–§10p — website transport, inventories
- `docs/invariants/backend.md` B25
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/meeting-live-map-mirror.mdc`
- `.cursor/rules/agenda-item-meeting-child.mdc`
- `.cursor/rules/decision-meeting-child.mdc`
- `.cursor/rules/vote-decision-child.mdc`
- `.cursor/rules/talk-record-meeting-child.mdc`
- `.cursor/rules/sequelize-include-by-association-name.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
