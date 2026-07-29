# Meeting Live State (CRDT)

## Scope

The collaborative state of a live meeting: the Yjs document that carries `subject`, `type`, `status`, `datetime`, and `participants` while the meeting runs, the `meetings.live_state` BLOB that persists it, and the in-memory registry that owns the document per process.

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
};

type MeetingLiveMap = {
    subject: string;
    type: MeetingLiveType;
    status: MeetingLiveStatus;
    datetime: string;            // ISO-8601 from SQL Meeting.datetime
    participants: Record<string, MeetingLiveParticipant>; // keyed by member id
};
```

Attendance is timestamp-only (`attendedAt` / `leftAt`); there is no separate boolean. Connection presence fields are seeded `OFFLINE` / null timestamps on first document create; writers that flip online/offline are **not** shipped yet (`meeting-realtime-socket.md` shipped limits).

`datetime` is the scheduled start used by the website client session gate for the self-check-in open window (`MeetingModel.ATTEND_OPEN_BEFORE_MS` — 30 minutes before start). It is seeded from SQL and is **not** a collaborative edit target.

### 1.3 Document codec and seed

File: `backend/src/app/helpers/MeetingLiveDocHelper.ts` (same module as the registry). Imports `MEETING_LIVE_MAP` / live types from the mirror (§1.2).

Codec/seed helpers are **module-private**. Public surface of this file is the registry (§2) plus `readLiveFields` (deferred column apply, §6).

| Member | Role |
|---|---|
| `createLiveDoc(fields)` | Private. One `transact`: sets `subject` / `type` / `status` / `datetime`; builds nested `participants` as `Y.Map` of per-id `Y.Map(Object.entries(participant))` |
| `encodeLiveDoc` / `decodeLiveDoc` | Private. V2 BLOB codec |
| `buildLiveParticipants(meeting)` | Private. `meeting.getParticipants({ include: [{ association: "member", required: true }] })` → `Record` keyed by `member_id`; skips a row only if `member` is somehow missing; ISO-maps `attended_at` / `left_at`; seeds `connectionStatus: "OFFLINE"` |
| `getLiveDoc(meeting)` | Private. Non-empty `live_state` → `decodeLiveDoc`; else `createLiveDoc` from SQL columns + `buildLiveParticipants` |
| `readLiveFields(doc)` | **Exported.** `doc.getMap(MEETING_LIVE_MAP).toJSON() as Partial<MeetingLiveMap>` — only sanctioned read-back |

Nested `Y.Map` for each participant is required so collaborative field updates (e.g. `connectionStatus`) merge by identity. A plain object stored under `participants` would not be a collaborative map.

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
7. **No automatic BLOB schema migration.** A non-empty `live_state` is decoded as-is. Documents created before `participants` / `datetime` existed are not back-filled on load; a requester reset (§3.1) clears the BLOB so the next sync re-seeds from SQL.

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

## 10) Related

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-domain.md` — columns, GQL read, requester write path, `ATTEND_OPEN_BEFORE_MS`
- `docs/platforms/website/organization-host-routing.md` §5.1, §5.3 — website transport, SyncedStore binding, session surface
- `docs/invariants/backend.md` B25
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/rules/meeting-live-map-mirror.mdc`
- `.cursor/rules/sequelize-include-by-association-name.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
