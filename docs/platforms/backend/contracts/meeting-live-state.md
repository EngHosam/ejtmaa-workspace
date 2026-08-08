# Meeting Live State (CRDT)

## Scope

The collaborative state of a live meeting: the Yjs document that carries `subject`, `type`, `status`, `datetime`, `minMembersCount`, `participants`, `currentTalkMemberId`, `agendaItems`, and `decisions` while the meeting runs, the `meetings.live_state` BLOB that persists it, and the in-memory registry that owns the document per process.

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
| `MeetingLiveAgendaItemStatuses` / `MeetingLiveAgendaItemStatus` | `WAITING` \| `DISCUSSING` \| `DONE` \| `CANCELED` | Agenda line status (SQL `agendaItemStatus` mirror) |
| `MeetingLiveAgendaItem` | see below | One agenda line in the live map |
| `MeetingLiveDecisionPhases` / `MeetingLiveDecisionPhase` | `PRE_START` \| `DURING` | Decision phase (SQL mirror) |
| `MeetingLiveDecisionStatuses` / `MeetingLiveDecisionStatus` | `NEW` \| `UNDER_VOTING` \| `ACCEPTED` \| `REJECTED` \| `CANCELED` | Decision status (SQL `decisionStatus` mirror) |
| `MeetingLiveDecisionVotingTypes` / `MeetingLiveDecisionVotingType` | `LIVE` \| `SECRET` | Optional voting type (SQL mirror) |
| `MeetingLiveVoteValues` / `MeetingLiveVoteValue` | `YES` \| `NO` | Ballot value (SQL `voteValue` mirror) |
| `MeetingLiveVote` | see below | One cast under a decision (`votes` key = member id) |
| `MeetingLiveDecision` | see below | One decision line (+ nested `votes` from SQL or empty) |
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
    status: MeetingLiveDecisionStatus;   // seeded from SQL (PRE_START NEW → UNDER_VOTING); live writers may change it
    votingType: MeetingLiveDecisionVotingType | null; // from SQL
    isLiveCreated: boolean;              // session-only
    votes: Record<string, MeetingLiveVote>; // keyed by member id; from SQL Vote at seed (may be empty)
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
    decisions: Record<string, MeetingLiveDecision>;       // keyed by decision id (all phases)
};
```

Attendance is timestamp-only (`attendedAt` / `leftAt`); there is no separate boolean. Connection presence fields are seeded `OFFLINE` / null timestamps on first document create; writers that flip online/offline are **not** shipped yet (`meeting-realtime-socket.md` shipped limits).

`talkTurn` is the live talk-queue order on the roster entry (`null` = not queued). Root `currentTalkMemberId` is who is speaking now (`null` = nobody). Both are session-only (not SQL `talk_records` columns); they die on session reset (§3.1). Durable queue/history remains `TalkRecord` (`talk-record-domain.md`). **Shipped client writers:** the website meeting talk queue — non-chair `actions.raiseHand` / `lowerHand` (own `talkTurn`) and chair `actions.giveTalkFloor` / `removeFromTalkQueue` / `endTalkFloor` (`currentTalkMemberId` + others' `talkTurn`), all live-map only with **no** `TalkRecord` write and no SQL reflect (`organization-host-routing.md` §5.3 / §5.5). Queue order is FIFO on ascending `talkTurn`; the values are allocated client-side (`max + 1`, skipping taken values) and are internal — clients renumber for display. Granting the floor clears the grantee's `talkTurn` in the same transaction, so a floor holder is never also queued. Existing non-empty BLOBs are decoded as-is (no talk-field backfill).

`datetime` is the scheduled start used by the website client session gate for the self-check-in open window (`MeetingModel.ATTEND_OPEN_BEFORE_MS` — 30 minutes before start). It is seeded from SQL and is **not** a collaborative edit target.

`minMembersCount` is the quorum denominator for the chair attendance log UI. It is seeded from SQL `min_members_count` on **first empty** `live_state` create only and is **not** a collaborative edit target. Existing non-empty BLOBs are decoded as-is (no scalar backfill); clients must treat a missing value as “hide quorum strip,” not invent a default.

`agendaItems` mirrors durable SQL `AgendaItem` lines (`id` / `sort_order` → `sortOrder` / `subject` / `status`) on **first empty** `live_state` create. Per-item `status` is seeded from SQL (`agendaItemStatus`). Session-only `isLiveCreated` defaults to `false`. Session cancel uses `status: "CANCELED"` (no live delete). Active agenda line is the one with `status: "DISCUSSING"` (no separate root pointer — do **not** add `currentAgendaItemId`). Durable `status` remains on SQL and is **not** auto-synced from live writes (§6). **Shipped client writer:** website chair `actions.setAgendaItemStatus` updates live-map `status` only (`organization-host-routing.md` §5.3 / §5.5). `isLiveCreated` writers and SQL reflect for agenda status are **not** shipped. Existing non-empty BLOBs are decoded as-is (no agenda backfill).

`decisions` mirrors durable SQL `Decision` rows (**both** `PRE_START` and `DURING`) on **first empty** `live_state` create: `sort_order` → `sortOrder`, `subject`, `phase`, `status`, `voting_type` → `votingType`. Session-only `isLiveCreated` defaults `false`. Per-decision `votes` seeds from SQL `Vote` rows keyed by `member_id` (`value`, `cast_at` → `castAt` ISO); no rows → empty nested collaborative `Y.Map` (logical `{}`). Do **not** pre-book vote keys for non-voters. Product: `PRE_START` = full list members may vote; `DURING` = chair advances one decision at a time by setting at most one decision to `UNDER_VOTING` (status is the active marker — no separate root pointer). Durable casts remain SQL `Vote` (`vote-domain.md`). Existing non-empty BLOBs decode as-is (no decision/votes backfill).

**PRE_START status normalization on seed.** `buildLiveDecisions` is the only place that rewrites a mirrored value: a `PRE_START` row whose SQL `status` is `NEW` **or** `UNDER_VOTING` is seeded as `UNDER_VOTING`. Pre-start ballots have no chair "open the vote" step — they are open from the moment the session document exists, so the live map carries the state the client acts on instead of asking every client to re-derive it. `DURING` rows and every other status (`ACCEPTED` / `REJECTED` / `CANCELED`) are mirrored unchanged, SQL is not written back, and an existing non-empty BLOB is never re-normalized (the row keeps whatever the document already holds).

**Shipped client writers (website, live map only, no SQL reflect).** `actions.castVote` adds `votes[memberId] = { memberId, value, castAt }` once per member on an `UNDER_VOTING` decision (a cast is final — the action refuses to overwrite an existing key). `actions.setDecisionStatus` opens a `DURING` vote (`NEW` → `UNDER_VOTING` + `votingType`, refused while another `DURING` decision is `UNDER_VOTING`) and settles an open one (`ACCEPTED` / `REJECTED` only with a strict majority, `CANCELED` always). `actions.clearDecisionVotes` empties `votes` on an open decision (revote). No durable `Vote` row is written by any of them (`vote-domain.md`); `isLiveCreated` writers remain unshipped. Gate matrix: `../../website/organization-host-routing.md` §5.3.

### 1.3 Document codec and seed

File: `backend/src/app/helpers/MeetingLiveDocHelper.ts` (same module as the registry). Imports `MEETING_LIVE_MAP` / live types from the mirror (§1.2).

Codec/seed helpers are **module-private**. Public surface of this file is the registry (§2), `readLiveFields`, and `completeMeetingLiveToSql` (§6).

| Member | Role |
|---|---|
| `createLiveDoc(fields)` | Private. One `transact`: nested `participants` then `currentTalkMemberId`; nested `agendaItems`; nested `decisions` (each with nested `votes` `Y.Map`, possibly empty) |
| `encodeLiveDoc` / `decodeLiveDoc` | Private. V2 BLOB codec |
| `buildLiveParticipants(meeting)` | Private. `meeting.getParticipants({ include: [{ association: "member", required: true }] })` → `Record` keyed by `member_id`; seeds `connectionStatus: "OFFLINE"`, `talkTurn: null` |
| `buildLiveAgendaItems(meeting)` | Private. `meeting.getAgendaItems()` → `Record` keyed by agenda `id`; maps SQL `status`; seeds `isLiveCreated: false` |
| `buildLiveDecisions(meeting)` | Private. `meeting.getDecisions({ include: [{ association: "votes" }] })` → `Record` keyed by decision `id` (all phases); maps SQL fields; seeds `isLiveCreated: false`; `votes` from SQL `Vote` by `member_id` (empty when none; no per-non-voter slots) |
| `getLiveDoc(meeting)` | Private. Non-empty `live_state` → `decodeLiveDoc` (no backfill); else `createLiveDoc` with participants / talk / agenda / decisions seeds (`currentTalkMemberId: null`) |
| `readLiveFields(doc)` | **Exported.** `doc.getMap(MEETING_LIVE_MAP).toJSON() as Partial<MeetingLiveMap>` — sanctioned read-back for complete reflect |
| `completeMeetingLiveToSql(meeting, fields, transaction?)` | **Exported.** Full durable reflect on complete (§6) |

Nested `Y.Map` for each participant, agenda item, decision, and per-decision `votes` map is required so collaborative field updates merge by identity. A plain object under those keys would not be a collaborative map. Seed must not store plain `{}` as the Yjs value of `votes` — use a nested `Y.Map` (logical seed input may be `votes: {}` or populated from SQL).

SQL column enums remain `MeetingType` / `MeetingStatus` from `G_Tr` keys; the CRDT document uses `MeetingLiveMap` only.

**Encoding is V2 on every hop** — BLOB, sync response, and update broadcast. A V1 update applied with `applyUpdateV2` (or the reverse) throws; the website converts its V1 doc events with `Y.convertUpdateFormatV1ToV2` before emitting.

`readLiveFields` is called from `MeetingLiveCompleteIOController` before `completeMeetingLiveToSql`.
## 2) Registry and persistence

File: `backend/src/app/helpers/MeetingLiveDocHelper.ts`.

Two module-level maps keyed by `meetingId`: `entries` (live documents) and `inflight` (in-progress loads). `PERSIST_DEBOUNCE_MS = 1500`.

| Export | Behavior |
|---|---|
| `getOrCreateMeetingLiveDoc(meetingId)` | Returns the cached entry when present and live; otherwise loads once. Throws `"MEETING_NOT_LIVE"` when SQL/`entry.meeting` status is outside `MEETING_LIVE_STATUSES` (blocks post-complete re-seed). Concurrent callers await the same `inflight` promise |
| `peekMeetingLiveDoc(meetingId)` | Returns the **registry entry reference** or `undefined`; never loads. Callers may rebind `entry.meeting` to the socket meeting instance |
| `schedulePersistMeetingLiveDoc(meetingId)` | Restarts the debounce timer |
| `flushPersistMeetingLiveDoc(meetingId)` | Clears the timer and awaits `meeting.update({ live_state: encodeLiveDoc(doc) })` |
| `destroyMeetingLiveDoc(meetingId, { flush })` | Flushes (default) or cancels the timer, then `doc.destroy()` and drops the entry |
| `completeMeetingLiveToSql(meeting, fields, transaction?)` | Durable complete reflect (§6) |

Load path inside `getOrCreateMeetingLiveDoc`:

1. `Meeting().findByPk(meetingId)`; a missing row throws `"404"`.
2. Status must be in `MEETING_LIVE_STATUSES`; otherwise throws `"MEETING_NOT_LIVE"`.
3. Records whether the row already had a non-empty `live_state`.
4. `getLiveDoc(meeting)` (§1.3) and stores `{ doc, meeting }` in `entries`.
5. Subscribes `doc.on("update")` → `schedulePersistMeetingLiveDoc`.
6. Schedules one persist when the row had **no** BLOB, so the seeded document reaches the column even if nobody edits.

The timer callback is not awaited by anyone, so a failed write is caught and reported through `Log().unhandledError(err)` instead of becoming an unhandled rejection.

The Sequelize row captured at load is the instance the flush writes through until a lifecycle controller rebinds `entry.meeting` to `socket.data.meeting` (start/complete). After rebind, one instance owns both the handshake row and the registry gate.
## 3) Lifecycle

1. A meeting is created and edited through `MeetingRequester`; SQL columns are the only truth and `live_state` stays `null` (`meeting-domain.md` §9).
2. First `meeting.live.sync` on the meeting: the registry seeds a document from `subject` / `type` / `status` and `buildLiveParticipants()`, then schedules the first BLOB write.
3. While the meeting is live, the document is the source of truth for those map fields. Every accepted update mutates the document, is broadcast to the room, and restarts the debounce.
4. The BLOB trails the document by up to the debounce window (§7.1).
5. Reconnect or a second participant replays the whole state through the sync handshake; nothing is read from the SQL columns again while the entry stays in memory.
6. Chair `meeting.live.start` writes SQL `status → STARTED` immediately (no Yjs mutate/rebroadcast from the server). Chair `meeting.live.complete` runs `completeMeetingLiveToSql` (§6), then destroys the registry entry after commit.

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

## 6) Applying live fields to SQL columns (complete reflect)

Product decision: while the session runs, collaborative map fields are **not** written back to SQL on every `meeting.live.update`. Durable reflect runs **once** on chair `meeting.live.complete` via `completeMeetingLiveToSql`.

**Immediate lifecycle SQL (outside complete):** `meeting.live.start` writes meeting `status → STARTED` only (B25 lifecycle exception). Live-map `status = "STARTED"` still fans out through the normal client `meeting.live.update` / Yjs path — the start controller must not encode or re-broadcast CRDT updates.

### 6.1 `completeMeetingLiveToSql`

Signature: `completeMeetingLiveToSql(meeting: MeetingModel, fields: MeetingLiveCompleteFields, transaction?)`.

`MeetingLiveCompleteFields` accepts only:

- `agendaItems`
- `decisions` (incl. nested `votes`)
- `participants` (attendance timestamps)

**Not applied from live:** meeting `subject` / `type`. Talk queue (`talkTurn` / `currentTalkMemberId`) and connection presence stay session-only (no `TalkRecord` write).

Transaction pattern (repo-native):

```ts
await Meeting().transaction({ transaction }, async transaction => {
  // all writes use { transaction }
});
```

Inside the transaction:

1. Rebind registry `entry.meeting = meeting` when an entry exists (one instance with the socket row).
2. `meeting.update({ status: "COMPLETED", live_state: null }, { transaction })`.
3. **AgendaItem:** for each live id — `findByPk` → update `sort_order` / `subject` / `status` when `meeting_id` matches; if missing and `isLiveCreated` → `build` + `setDataValue("id", liveId)` + `save`.
4. **Decision:** same pattern for `sort_order` / `subject` / `phase` / `status` / `voting_type`.
5. **Vote:** per decision, make SQL match the live `votes` map exactly (destroy SQL votes absent from live; upsert present by composite PK).
6. **MeetingParticipant:** for each live roster key, update `attended_at` / `left_at` (ISO → `Date` / null); skip unknown member ids.
7. `transaction.afterCommit(() => destroyMeetingLiveDoc(meetingId, { flush: false }))`.

### 6.2 After complete

- Registry entry is gone; `getOrCreateMeetingLiveDoc` refuses non-live status (`"MEETING_NOT_LIVE"`).
- Sync/update/complete map that throw to `rejectLive("MEETING_NOT_LIVE")`.
- No outbound `meeting.live.completed` event — clients learn `COMPLETED` from the Yjs fan-out written by the chair before/with the complete emit.

Until complete, GraphQL readers still see requester SQL values for collaborative fields (except `status` after start).

## 7) Shipped limits (intentional)

1. **Debounce has no maximum wait.** Every document update restarts the 1500 ms timer, so a continuously edited meeting keeps deferring its BLOB write until the editing pauses.
2. **No flush on shutdown.** Nothing hooks process exit; unflushed changes are lost on restart.
3. **Stale status gate on `meeting.live.update`.** The update gate still reads `socket.data.meeting` from the handshake (or the rebound instance after start/complete on that socket). Other sockets keep their handshake snapshot until reconnect; both `WAITING_TO_START` and `STARTED` remain in `MEETING_LIVE_STATUSES`, so peer updates still pass after start. Post-complete, `getOrCreate` refuses non-live SQL status even if a peer handshake snapshot is still live.
4. **Eviction on approve / demotion / complete.** `destroyMeetingLiveDoc` runs from requester reset (§3.1) and from complete reflect `afterCommit` (§6). Nothing else evicts when a session merely disconnects.
5. **Single instance.** The registry is process memory. Two backend processes would each hold an independent document for the same meeting and overwrite each other's BLOB; horizontal scaling needs a shared document plane first.
6. **Seed write on first load.** A meeting whose BLOB is `null` gets one write on first sync even when nobody edits. Harmless because a session only opens for a live meeting, whose columns are no longer edited through the form.
7. **No automatic BLOB schema migration.** A non-empty `live_state` is decoded as-is. Documents created before `participants` / `datetime` / `minMembersCount` / `agendaItems` / `talkTurn` / `currentTalkMemberId` / `decisions` / nested `votes` existed are not back-filled on load; a requester reset (§3.1) or complete (§6) clears the BLOB so the next eligible sync re-seeds from SQL.
## 8) Traceability

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Meeting.ts` | `live_state` column only | §1.1 |
| `backend/src/app/types/meeting.ts` | mirrored live map contract (`MeetingLiveMap`, participants, `MEETING_LIVE_*`) | §1.2; `.cursor/rules/meeting-live-map-mirror.mdc` |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | private codec+seed + registry + `readLiveFields` + `completeMeetingLiveToSql` | §1.3, §2, §6 |
| `backend/src/app/orm/models/Customer.ts` | Ability notify-template roster `include: ["member"]` | §1.3; `meeting-domain.md` §9.1 |
| `backend/src/app/orchestrator/requesters/MeetingRequester.ts` | `live_state = null` + `afterCommit` destroy on approve and on demotion to draft | §3.1 |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | `registerOrmAttrs.expect: ["live_state"]` | §4 |
| `backend/eng-hosam/@nodejs/gql/src/BridgeBase.ts` | `bootRegisteredAttrs()` exclusion semantics (library, not edited) | §4 |
| `backend/package.json` | `yjs` dependency pin | Scope |
| `backend/src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | reads the registry document; maps `"MEETING_NOT_LIVE"` | `meeting-realtime-socket.md` §3 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | writes the registry document; enforces `MEETING_LIVE_STATUSES`; maps `"MEETING_NOT_LIVE"` | `meeting-realtime-socket.md` §3–§4 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveStartIOController.ts` | chair SQL `STARTED`; bind `entry.meeting` | `meeting-realtime-socket.md` §3.4 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveCompleteIOController.ts` | chair complete → `completeMeetingLiveToSql` | `meeting-realtime-socket.md` §3.5; §6 |

## 9) Change set inventory (durable agenda/decision enums + live SQL seed)

**Current ship.** Durable `AgendaItem.status` (`agendaItemStatus`) and `decisionStatus.CANCELED`; customer GQL mirrors; first empty `live_state` seeds agenda `status` and decision `votes` from SQL. Session-only live fields: `isLiveCreated` (agenda/decision), `talkTurn`, `currentTalkMemberId`. Active agenda = `status: "DISCUSSING"`; active DURING decision = `status: "UNDER_VOTING"` — **no** root `currentAgendaItemId` / `currentDecisionId`. Live map contract + seed order: §§1.2–1.3. **Shipped website live writers:** chair `setAgendaItemStatus` (map `status` only), the talk-queue set `raiseHand` / `lowerHand` / `giveTalkFloor` / `removeFromTalkQueue` / `endTalkFloor` (`talkTurn` + `currentTalkMemberId` only), and the decisions set `castVote` / `setDecisionStatus` / `clearDecisionVotes` (decision `status` / `votingType` / nested `votes` only) — none reflect to SQL, talk turns write no `TalkRecord` row, and casts write no `Vote` row. **Not shipped:** `isLiveCreated` live writers (create path ready on complete). Talk queue / TalkRecord still not reflected on complete.

### Backend

| Path | State | Where described |
|---|---|---|
| `src/app/orm/models/AgendaItem.ts` | modified — `status` column / `AgendaItemStatus` | `agenda-item-domain.md` §3 |
| `src/resources/trans/ar/general.ts` | modified — `agendaItemStatus`; `decisionStatus.CANCELED`; `DONE` label `مكتمل` | enums |
| `src/resources/trans/en/general.ts` | modified — same keys EN | enums |
| `src/app/gql/definitions/base.graphql` | modified — `_AgendaItemStatus*`; `_DecisionStatusValue.CANCELED` | `graphql-and-types.md` |
| `src/app/gql/definitions/customer.graphql` | modified — `_AgendaItem.status` | `agenda-item-domain.md` §4 |
| `src/app/gql/gql-types/base.ts` | generated | excluded from narrative |
| `src/app/gql/gql-types/customer.ts` | generated | excluded from narrative |
| `src/app/gql/gql-types/supervisor.ts` | generated (regen) | excluded from narrative |
| `src/app/orchestrator/requesters/MeetingRequester.ts` | modified — `createAgendaItem` → `status: "WAITING"` | `agenda-item-domain.md` |
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — agenda status from SQL; decisions include `votes` from SQL | §1.3 |
| `src/app/types/meeting.ts` | live map already includes decisions/votes / agenda status unions (committed mirror) | §1.2 |

### Website

| Path | State | Where described |
|---|---|---|
| `src/types/gql/definitions/base.graphql` | mirrored | `graphql-mirror-and-tooling.md` |
| `src/types/gql/definitions/customer.graphql` | mirrored | same |
| `src/types/gql/gql-types/base.ts` | mirrored generated | excluded from narrative |
| `src/types/gql/gql-types/customer.ts` | mirrored generated | excluded from narrative |
| `src/types/meeting.ts` | identical live map mirror (committed) | §1.2 |
| `lib/tsconfig.tsbuildinfo` | type-check cache | **excluded** |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-live-state.md` | this page | — |
| `docs/platforms/backend/contracts/agenda-item-domain.md` | durable `status` | agenda contract |
| `docs/platforms/backend/contracts/decision-domain.md` | `CANCELED` + live seed note | decision contract |
| `docs/platforms/backend/contracts/vote-domain.md` | live nest seeds SQL votes | vote contract |
| `docs/platforms/backend/contracts/talk-record-domain.md` | live talk fields vs SQL | talk contract |
| `docs/platforms/backend/contracts/graphql-and-types.md` | `_AgendaItemStatus` index | GQL index |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | plane table | planes |
| `docs/platforms/backend/README.md` | index blurbs | backend index |
| `docs/README.md` | live-state index blurb | root index |
| `docs/platforms/website/organization-host-routing.md` | §5.3–§5.5 / §8 / §10 (agenda UI, overlay frost, session action, LiveKit enterLive) | website |
| `docs/platforms/website/README.md` | inventory pointer → §9 | website index |
| `.cursor/rules/meeting-live-state.mdc` | seed/order invariants | governance |
| `.cursor/rules/meeting-live-map-mirror.mdc` | mirror + `isLiveCreated` only | governance |
| `.cursor/rules/agenda-item-meeting-child.mdc` | durable status | governance |
| `.cursor/rules/decision-meeting-child.mdc` | `CANCELED` + live seed | governance |
| `.cursor/rules/vote-decision-child.mdc` | live votes from SQL | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | 6c / 6e seed checklist | skill |

### Triage

| Symptom | Likely cause |
|---|---|
| Client missing `decisions` / nested `votes` / agenda `status` after sync | Non-empty pre-change `live_state` BLOB (no backfill). Reset via approve/demote (§3.1) |
| `votes` empty after seed | No SQL `Vote` rows — expected |
| Agenda GQL `status` missing | Client selection set omits `status` |
| Expecting per-member empty vote slots | Rejected — keys only for existing SQL votes or later writers |

## 11) Change set inventory (live start + complete SQL reflect)

**Current ship.** Dedicated `/meeting` lifecycle events `meeting.live.start` / `meeting.live.complete` (chair-only). Immediate SQL `STARTED` on start; full durable reflect + `live_state = null` + registry destroy on complete (`completeMeetingLiveToSql`). Website `startMeeting` / `endMeeting` emit those events and batch live status. Init product CTA for end is documented in `organization-host-routing.md` §5.5 / §10s. B25 amended for this narrow lifecycle path. Behavior: §§2–3, §6; transport: `meeting-realtime-socket.md` §3.4–§3.5; website: `organization-host-routing.md` §5.3, §5.5.

### Backend (`backend/`)

| Path | State | Where described |
|---|---|---|
| `src/app/helpers/MeetingLiveDocHelper.ts` | modified — `MeetingLiveCompleteFields`, `completeMeetingLiveToSql`, non-live `getOrCreate` gate, peek returns entry ref | §2, §6 |
| `src/app/socket/controllers/meeting/MeetingLiveStartIOController.ts` | **added** | `meeting-realtime-socket.md` §3.4 |
| `src/app/socket/controllers/meeting/MeetingLiveCompleteIOController.ts` | **added** | `meeting-realtime-socket.md` §3.5 |
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | modified — bound events + chair-only compose + `NOT_CHAIR` | `meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | modified — map `"MEETING_NOT_LIVE"` | `meeting-realtime-socket.md` §3.1 |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | modified — map `"MEETING_NOT_LIVE"` (still no SQL) | `meeting-realtime-socket.md` §3.2 |
| `src/resources/configs/socket/io.ts` | modified — register start/complete | `meeting-realtime-socket.md` §1 |

### Website (`website/`)

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useMeetingLive.tsx` | modified — `emitLiveStart` / `emitLiveComplete`, `socketRef`, `NOT_CHAIR` | `organization-host-routing.md` §5.1, §5.3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | modified — start/end emit + batch | `organization-host-routing.md` §5.3 |
| `lib/tsconfig.tsbuildinfo` | type-check cache | **excluded** (generated) |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/invariants/backend.md` | B25 lifecycle amendment | B25 |
| `docs/platforms/backend/contracts/meeting-live-state.md` | this page §6 shipped | — |
| `docs/platforms/backend/contracts/meeting-realtime-socket.md` | start/complete events | socket contract |
| `docs/platforms/website/organization-host-routing.md` | §5.3 actions; §5.5 / §10s Init end CTA | website |
| `.cursor/rules/meeting-live-state.mdc` | reflect + lifecycle exception | governance |
| `.cursor/rules/meeting-realtime-socket.mdc` | events + chair bind | governance |
| `.cursor/rules/website-meeting-live-session.mdc` | start/end emit | governance |
| `.cursor/skills/meeting-realtime-socket/SKILL.md` | lifecycle checklist | skill |
| `.cursor/skills/website-meeting-live-session/SKILL.md` | start/end emit | skill |

## 10) Related

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-domain.md` — columns, GQL read, requester write path, `ATTEND_OPEN_BEFORE_MS`
- `docs/platforms/backend/contracts/agenda-item-domain.md` — durable SQL agenda incl. `status`; live mirror + session fields
- `docs/platforms/backend/contracts/decision-domain.md` — durable SQL decisions incl. `CANCELED`; live `decisions` / nested `votes`
- `docs/platforms/backend/contracts/vote-domain.md` — durable SQL casts; live nest under `decisions[*].votes` (seeded from SQL)
- `docs/platforms/backend/contracts/talk-record-domain.md` — durable SQL talk queue/history; live `talkTurn` / `currentTalkMemberId`
- `docs/platforms/backend/contracts/livekit-media-plane.md` — LiveKit is not agenda/talk/decision/vote truth
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.3, §8 — website transport / live field list
- `docs/invariants/backend.md` B25
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/meeting-live-map-mirror.mdc`
- `.cursor/rules/agenda-item-meeting-child.mdc`
- `.cursor/rules/decision-meeting-child.mdc`
- `.cursor/rules/vote-decision-child.mdc`
- `.cursor/rules/talk-record-meeting-child.mdc`
- `.cursor/rules/sequelize-include-by-association-name.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
