# TalkRecord Domain Contract (Current)

## 1) Scope

Current Ejtmaa talk-record surface:

- ORM persistence for durable talk-queue / floor-history rows linked to a `Meeting` (and a speaking `Member`),
- customer GraphQL **nested** read under `_Meeting.talkRecords`,
- nested `member` on each talk record,
- website GQL mirrors for that nested surface.

Out of scope (not shipped):

- root queries `talkRecords` / `talkRecord(...)`,
- GraphQL `meeting` on `_TalkRecord` (parent meeting supplies context),
- FK scalars on `_TalkRecord` (`meeting_id`, `member_id`),
- `decision_id` / any Decision link (product: queue is meeting-scoped only),
- talk-record write requesters / mutations,
- supervisor TalkRecord GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for talk records,
- LiveKit A/V coupling (media is separate; this model is durable SQL queue/history only).

## 2) Domain purpose

`TalkRecord` is a **non-actor** durable queue entry: one member’s turn (waiting / speaking / completed / canceled) inside one meeting.

- Identity: surrogate UUID `id`.
- Scoped by `meeting_id`; speaker by `member_id`.
- Explicit product decision: **no** `decision_id` — talk queue is not decision-scoped.
- GraphQL nest path: `_Meeting.talkRecords` → `_TalkRecord.member`.
- Expose **`member` only** among relations on `_TalkRecord`; do **not** nest `meeting` while the parent path already is the meeting.

## 3) ORM model

File: `backend/src/app/orm/models/TalkRecord.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `talkRecord` (default `hasMany` from Meeting → `talkRecords` — **no** `as`)
- `tableName`: `talk_records`

### 3.1 Attrs layout

- `//relations` — `meeting_id`, `member_id`
- `//info` — `sort_order`, `status`, `started_at`, `ended_at`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID PK | no | default `UUIDV4` |
| `meeting_id` | UUID | no | FK → Meeting.id |
| `member_id` | UUID | no | FK → Member.id |
| `sort_order` | INTEGER | yes | default `null` |
| `status` | STRING(191) | no | enum `talkRecordStatus`; default `WAITING` |
| `started_at` | DATE | no | when the record / turn started (or was enqueued — as written by writers) |
| `ended_at` | DATE | yes | default `null` |

Exported TS type: `TalkRecordStatus`.

### 3.3 Enums (localized)

| Enum key | Values |
|---|---|
| `talkRecordStatus` | `WAITING`, `SPEAKING`, `COMPLETED`, `CANCELED` |

AR/EN: `backend/src/resources/trans/ar/general.ts`, `en/general.ts`.

### 3.4 Indexes

- `talk_records_meeting_id`
- `talk_records_member_id`

### 3.5 Relations

`TalkRecord.boot()`:

- `belongsTo(Meeting)` on `meeting_id`
- `belongsTo(Member)` on `member_id`

`Meeting.boot()`:

```ts
this.hasMany(TalkRecord(), {
    sourceKey: "id",
    foreignKey: "meeting_id"
});
```

No `as`. Mixins on Meeting: `talkRecords` / `getTalkRecords` / `createTalkRecord` / ….

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_TalkRecordStatus` / `_TalkRecordStatusValue`
- `backend/src/app/gql/definitions/customer.graphql` — `_TalkRecord` + `_Meeting.talkRecords`

### Type `_TalkRecord`

Implements `_Timestamps` & `_Pagination`.

Info: `id`, `sort_order`, `status`, `started_at`, `ended_at`.

Relations: `member: _Member`.

Timestamps / pagination: `created_at`, `updated_at`, `total_count`.

**Not exposed:** `meeting_id`, `member_id`, `meeting`, `decision`.

### Nested under `_Meeting`

```graphql
talkRecords: [_TalkRecord]
```

Cardinality: talk-queue rows per meeting expected within board-session size (B15 OK under meeting nest).

### Root queries

**None.**

### Bridge: `TalkRecordBridge`

File: `backend/src/app/gql/bridges/customer/TalkRecordBridge.ts`

- Extends `CustomerBridgeBase`
- `ident = "talkRecords"`
- `typeIdent = "_TalkRecord"`
- `ormModel = TalkRecordModel`
- `GetManyParent = MeetingModel`
- `GetOneParent = TalkRecordModel`

### Nested parent typing

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Meeting.talkRecords` | `TalkRecordBridge` | `GetManyParent = MeetingModel` |
| `_TalkRecord.member` | `MemberBridge` | `GetOneParent` includes `TalkRecordModel` |

### Registered bridges

`CustomerSchema.registeredBridges` includes `TalkRecordBridge`.

## 5) Read flow (nested)

```text
meeting(id) {
  talkRecords {
    id
    sort_order
    status { value label }
    started_at
    ended_at
    member { id name }
  }
}
```

1. `MeetingBridge` (org-scoped root)
2. `TalkRecordBridge` with parent Meeting (`talkRecords` association)
3. `MemberBridge` with parent TalkRecord

## 6) Seed

No talk-record seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — base + customer include `_TalkRecordStatus`, `_TalkRecord`, `_Meeting.talkRecords` |
| `cpanel/` | Deferred |

Verification: `yarn generate-types`, `yarn type-check` in `backend/`; copy base + customer SDL/types to `website/src/types/gql/**`.

## 8) Failure modes (read path)

Nested talk records inherit meeting read gates; no separate root failure modes.

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/TalkRecord.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Meeting.ts` | `hasMany` TalkRecord + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | `talkRecordStatus` AR | §3.3 |
| `backend/src/resources/trans/en/general.ts` | `talkRecordStatus` EN | §3.3 |
| `backend/src/app/gql/definitions/base.graphql` | `_TalkRecordStatus` | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_TalkRecord` + `_Meeting.talkRecords` | §4 |
| `backend/src/app/gql/bridges/customer/TalkRecordBridge.ts` | nested bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | `GetOneParent` + `TalkRecordModel` | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register bridge | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enum); no talk-record roots | §7 |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry `TalkRecord` (gitignored) | excluded |
| `.cursor/skills/orm-model-generator/SKILL.md` | ORM generator note | governance |
| `.cursor/skills/gql-schema-bridge-generator/SKILL.md` | GQL generator note | governance |
| `.cursor/rules/talk-record-meeting-child.mdc` | Durable child invariants | governance |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `.cursor/rules/talk-record-meeting-child.mdc`
- `.cursor/rules/agenda-item-meeting-child.mdc` (naming preference for default plural associations)
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
