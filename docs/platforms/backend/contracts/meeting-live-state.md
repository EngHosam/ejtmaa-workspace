# Meeting Live State (CRDT)

## Scope

The collaborative state of a live meeting: the Yjs document that carries `subject`, `type`, and `status` while the meeting runs, the `meetings.live_state` BLOB that persists it, and the in-memory registry that owns the document per process.

Transport (namespace, handshake, events, authorization): `docs/platforms/backend/contracts/meeting-realtime-socket.md`.
Website consumer (`useLiveMeeting`, SyncedStore): `docs/platforms/website/organization-host-routing.md` §5.1.
Meeting domain (columns, GQL, requester write path): `docs/platforms/backend/contracts/meeting-domain.md`.

Dependency: `yjs@13.6.27` (`backend/package.json`), same major/minor as the website copy.

## 1) Model surface

File: `backend/src/app/orm/models/Meeting.ts`, section `live session` (after `boot()`, before the relation mixins).

### 1.1 Column

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `live_state` | `BLOB` | yes | `null` | Yjs V2 state snapshot of the whole document; `null` until the first persist |

Declared in the `Attrs` type under a `//live session` comment group, separate from `//info`.

### 1.2 Statics and instance helper

| Member | Signature | Behavior |
|---|---|---|
| `LIVE_MAP` | `"meeting"` | Name of the `Y.Map` holding the live fields. The website SyncedStore root key must be the same string, otherwise the two sides sync disjoint maps |
| `LIVE_STATUSES` | `Array<MeetingStatus>` = `["WAITING_TO_START", "STARTED"]` | The only statuses that accept live writes (`meeting-realtime-socket.md` §4) |
| `createLiveDoc(fields)` | `(MeetingLiveFields) => Y.Doc` | New doc seeded in a single `transact` with `subject`, `type`, `status` |
| `encodeLiveDoc(doc)` | `(Y.Doc) => Buffer` | `Y.encodeStateAsUpdateV2` → `Buffer` for the BLOB |
| `decodeLiveDoc(liveState)` | `(Buffer \| Uint8Array \| null \| undefined) => Y.Doc` | Empty doc when the input has no length; otherwise `Y.applyUpdateV2` on `new Uint8Array(liveState)` — the wrap is what makes a Sequelize `Buffer` assignable to the `Uint8Array` parameter |
| `readLiveFields(doc)` | `(Y.Doc) => Partial<MeetingLiveFields>` | Reads the three keys out of `LIVE_MAP` |
| `getLiveDoc()` | instance → `Y.Doc` | `live_state` present and non-empty → `decodeLiveDoc`; otherwise `createLiveDoc` seeded from the SQL columns |

Exported type: `MeetingLiveFields = { subject: string; type: MeetingType; status: MeetingStatus }`.

**Encoding is V2 on every hop** — BLOB, sync response, and update broadcast. A V1 update applied with `applyUpdateV2` (or the reverse) throws; the website converts its V1 doc events with `Y.convertUpdateFormatV1ToV2` before emitting.

`readLiveFields` has **no caller today**. It exists for the deferred column apply (§6) and is the only sanctioned way to read the live fields back out.

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
3. `meeting.getLiveDoc()` (§1.2) and stores `{ doc, meeting }` in `entries`.
4. Subscribes `doc.on("update")` → `schedulePersistMeetingLiveDoc`.
5. Schedules one persist when the row had **no** BLOB, so the seeded document reaches the column even if nobody edits.

The timer callback is not awaited by anyone, so a failed write is caught and reported through `Log().unhandledError(err)` instead of becoming an unhandled rejection.

The Sequelize row captured at load is the same instance the flush writes through. It is a **snapshot**: columns changed elsewhere afterwards are not reflected in it.

## 3) Lifecycle

1. A meeting is created and edited through `MeetingRequester`; SQL columns are the only truth and `live_state` stays `null` (`meeting-domain.md` §9).
2. First `meeting.live.sync` on the meeting: the registry seeds a document from `subject` / `type` / `status` and schedules the first BLOB write.
3. While the meeting is live, the document is the source of truth for those three fields. Every accepted update mutates the document, is broadcast to the room, and restarts the debounce.
4. The BLOB trails the document by up to the debounce window (§7.1).
5. Reconnect or a second participant replays the whole state through the sync handshake; nothing is read from the SQL columns again while the entry stays in memory.
6. Applying the live fields back onto the SQL columns is **not shipped** (§6).

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

Product decision: `subject`, `type`, and `status` are **not** written back to the meeting columns while the session runs. The reflection happens once when the meeting is completed, and that step is not implemented in this change set.

What it will own when it lands:

- read the live values through `Meeting().readLiveFields(doc)`,
- write the columns and the terminal `status` atomically with whatever else completion touches,
- flush and evict the registry entry (`destroyMeetingLiveDoc`) so no writer keeps mutating a completed meeting,
- close the stale-status window described in §7.3.

Until then the SQL columns keep the values the requester write path left, and consumers that read a meeting through GraphQL see those, not the live edits.

## 7) Shipped limits (intentional)

1. **Debounce has no maximum wait.** Every document update restarts the 1500 ms timer, so a continuously edited meeting keeps deferring its BLOB write until the editing pauses.
2. **No flush on shutdown.** Nothing hooks process exit; unflushed changes are lost on restart.
3. **Stale status gate.** The update gate reads the meeting row captured at handshake time (`socket.data.meeting`), so a status change made outside the live session is not seen until the socket reconnects. Acceptable because completion runs through the live session itself (§6).
4. **No eviction.** `destroyMeetingLiveDoc` and `peekMeetingLiveDoc` have no callers, so entries live until the process restarts and memory grows with the number of meetings touched.
5. **Single instance.** The registry is process memory. Two backend processes would each hold an independent document for the same meeting and overwrite each other's BLOB; horizontal scaling needs a shared document plane first.
6. **Seed write on first load.** A meeting whose BLOB is `null` gets one write on first sync even when nobody edits. Harmless because a session only opens for a live meeting, whose columns are no longer edited through the form.

## 8) Traceability

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Meeting.ts` | `live_state` column, `LIVE_MAP`, `LIVE_STATUSES`, doc statics, `getLiveDoc()` | §1 |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | registry, debounce persist, flush, destroy | §2 |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | `registerOrmAttrs.expect: ["live_state"]` | §4 |
| `backend/eng-hosam/@nodejs/gql/src/BridgeBase.ts` | `bootRegisteredAttrs()` exclusion semantics (library, not edited) | §4 |
| `backend/package.json` | `yjs` dependency pin | Scope |
| `backend/src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | reads the registry document | `meeting-realtime-socket.md` §3 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | writes the registry document, enforces `LIVE_STATUSES` | `meeting-realtime-socket.md` §3–§4 |

## 9) Change set inventory

Every tracked path of this delivery, in both repositories. Website paths are described in `docs/platforms/website/organization-host-routing.md` §5.1, §5.2, §10.

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/orm/models/Meeting.ts` | modified | §1; `meeting-domain.md` §3.2, §3.6 |
| `src/app/helpers/MeetingLiveDocHelper.ts` | added | §2 |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | modified | §4 |
| `src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | added | `meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | added | `meeting-realtime-socket.md` §3.1 |
| `src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | added | `meeting-realtime-socket.md` §3.2 |
| `src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | modified — now extends the base, returns the live set | `meeting-realtime-socket.md` §3 |
| `src/app/socket/controllers/meeting/MeetingJoinIOController.ts` | **deleted** — the log-only `meeting.join` probe it served no longer exists | `meeting-realtime-socket.md` §8 |
| `src/resources/configs/socket/io.ts` | modified — `meeting_join` replaced by the two live aliases and routes | `meeting-realtime-socket.md` §1 |
| `package.json` | modified — `yjs@13.6.27` | Scope |
| `yarn.lock` | modified — lock entries for the dependency above; not narrated | — |

### `website/`

| Path | State | Where described |
|---|---|---|
| `src/app/ui/components/meeting/hooks/useLiveMeeting.ts` | added | `organization-host-routing.md` §5.1 |
| `src/app/ui/components/meeting/LiveMeetingProbeScreen.tsx` | added | `organization-host-routing.md` §5.2 |
| `src/app/ui/components/meeting/hooks/useMeetingSocket.ts` | **deleted** — the socket-only hook is absorbed into `useLiveMeeting`; a separate session hook is now forbidden | `organization-host-routing.md` §5.1; `.cursor/rules/meeting-realtime-socket.mdc` |
| `src/app/ui/pages/Meeting.tsx` | modified — renders `LiveMeetingProbeScreen` instead of owning socket state | `organization-host-routing.md` §5, §5.2 |
| `package.json` | modified — `yjs`, `@syncedstore/core`, `@syncedstore/react` exact pins | `organization-host-routing.md` §5.1 |
| `yarn.lock` | modified — lock entries for the dependencies above; not narrated | — |
| `lib/tsconfig.tsbuildinfo` | modified — incremental build cache produced by `yarn type-check`; not behavioral | — |

Untracked build output under `backend/lib/`, `backend/.exporters/`, `backend/.types/`, `backend/.webpack_root.ts` and `website/server/` is generated by the build scripts and is not part of the source contract.

## 10) Related

- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — namespace, events, authorization
- `docs/platforms/backend/contracts/meeting-domain.md` — columns, GQL read, requester write path
- `docs/platforms/website/organization-host-routing.md` §5.1 — website session and SyncedStore binding
- `docs/invariants/backend.md` B25
- `.cursor/rules/meeting-live-state.mdc`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
