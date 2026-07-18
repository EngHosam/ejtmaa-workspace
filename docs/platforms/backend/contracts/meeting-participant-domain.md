# MeetingParticipant Domain Contract (Current)

## 1) Scope

Current Ejtmaa meeting-roster surface:

- ORM join row: who is invited/participating in a given `Meeting`,
- participant `type` (not a free-form permissions JSONB column),
- invite delivery tracking (`notified`, `delivery_status`, optional `first_online_at`),
- customer GraphQL **nested** read under `_Meeting.participants`,
- website GQL mirrors for that nested surface.

Out of scope (not shipped):

- root queries `meetingParticipants` / `meetingParticipant(...)` (explicitly declined),
- surrogate UUID `id` on the join row (composite PK kept),
- `belongsToMany` Meeting ↔ Member (removed; `hasMany` join rows only),
- GraphQL inverse `_Member.meetingParticipants` / `_Member.meetings` (not needed yet),
- GraphQL inverse `_MeetingParticipant.meeting` (SDL exposes `member` only today),
- permission helper implementation in code (matrix agreed; not a shipped module yet),
- participant write requesters / mutations,
- supervisor MeetingParticipant GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for participants,
- LiveKit / Yjs session planes.

## 2) Domain purpose

`MeetingParticipant` is a **non-actor** roster join between `Meeting` and `Member`.

- Profiles stay on `Member`; this row is meeting-scoped participation + invite delivery state.
- Permissions are **derived from `type`** (fixed matrix in product contract §8) — not stored per row.
- Tenant isolation is inherited via `Meeting.organization_id` (member must be same org; enforce on write paths when they ship).

Primary identity: composite primary key `(meeting_id, member_id)` — one roster row per member per meeting.

## 3) ORM model

File: `backend/src/app/orm/models/MeetingParticipant.ts`

Classification: **non-actor** (`Model<Attrs, Attrs>` — no `Ability`, no `can()`, no surrogate `id` in Attrs).

Persistence names:

- `modelName`: `meetingParticipant`
- `tableName`: `meeting_participants`

### 3.1 Attrs layout

- `//relations` — `meeting_id`, `member_id`
- `//info` — `type`, `notified`, `delivery_status`, `first_online_at`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `meeting_id` | UUID | no | PK part; FK → Meeting.id |
| `member_id` | UUID | no | PK part; FK → Member.id |
| `type` | STRING(191) | no | enum `meetingParticipantType` |
| `notified` | BOOLEAN | no | default `false` |
| `delivery_status` | STRING(191) | no | enum `meetingParticipantDeliveryStatus`; default `PENDING` |
| `first_online_at` | DATE | yes | first online timestamp; default `null` |

Exported TS types: `MeetingParticipantType`, `MeetingParticipantDeliveryStatus` from `G_Tr` enum keys.

No surrogate `id` column (product decision: composite key is sufficient for this join).

### 3.3 Enums (localized)

Under `backend/src/resources/trans/ar/general.ts` and `en/general.ts`:

| Enum key | Values |
|---|---|
| `meetingParticipantType` | `CHAIRPERSON`, `MEMBER`, `VIEWER` |
| `meetingParticipantDeliveryStatus` | `PENDING`, `SENT`, `FAILED` |

### 3.4 Indexes

None beyond the composite primary key in this change set.

### 3.5 Relations

`MeetingParticipant.boot()`:

- `belongsTo(Meeting)` on `meeting_id`
- `belongsTo(Member)` on `member_id`

`Meeting.boot()`:

- `hasMany(MeetingParticipant, { as: "participants" })` on `meeting_id`  
  (association `as` matches GQL bridge `ident` / SDL field `participants`)

`Member.boot()`:

- `hasMany(MeetingParticipant)` on `member_id`  
  (default Sequelize name → mixins `meetingParticipants` / `getMeetingParticipants` / …)

**Forbidden for this domain:** `Meeting.belongsToMany(Member)` / `Member.belongsToMany(Meeting)` through this join. Traversal is always via the join model.

Mixin declare blocks on MeetingParticipant: `meeting` / `member` (`BelongsTo*`).

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MeetingParticipantType`, `_MeetingParticipantDeliveryStatus` (+ Value enums)
- `backend/src/app/gql/definitions/customer.graphql` — `_MeetingParticipant` + `_Meeting.participants`

### Type `_MeetingParticipant`

Implements `_Timestamps` & `_Pagination`.

Info: `type`, `notified`, `delivery_status`, `first_online_at`.

Timestamps: `created_at`, `updated_at`.

Relations:

- `member: _Member`

Pagination: `total_count`.

**Not exposed as scalars:** `meeting_id`, `member_id`, surrogate `id`.

**Not exposed yet:** `meeting: _Meeting` (inverse nest deferred).

### Nested under `_Meeting`

```graphql
participants: [_MeetingParticipant]
```

Cardinality gate (B15): meeting roster is expected to stay well under 100 for board/assembly meetings → nested list is allowed. Organization-scale directories stay on roots.

### Root queries

**None.** No `meetingParticipants` / `meetingParticipant` on `Query` (explicit product decision).

### Bridge: `MeetingParticipantBridge`

File: `backend/src/app/gql/bridges/customer/MeetingParticipantBridge.ts`

- Extends `CustomerBridgeBase` (not org-owned root base — no root `me` path)
- `ident = "participants"` (must match Meeting association `as: "participants"`)
- `typeIdent = "_MeetingParticipant"`
- `ormModel = MeetingParticipantModel`
- `GetManyParent = MeetingModel`
- `GetOneParent = MeetingParticipantModel`
- No `getRootOrmParent` / `getOrmFindOptions` override

### Inverse / nested parent typing (mandatory)

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Meeting.participants` | `MeetingParticipantBridge` | `GetManyParent = MeetingModel` |
| `_MeetingParticipant.member` | `MemberBridge` | `GetOneParent` includes `MeetingParticipantModel` |

Current `MemberBridge` shape:

```ts
export type GetOneParent =
    | MemberModel
    | MeetingModel
    | MeetingParticipantModel
    | { me: true; id: string };
```

### Registered bridges

`CustomerSchema.registeredBridges` includes `MeetingParticipantBridge` (required for nested `participants` + `member` resolution). No new Query resolvers were added for this domain.

## 5) Read flow (nested)

### `meeting(id) { participants { … member { … } } }`

1. Root `MeetingBridge` loads the meeting (org-scoped via `{ me: true, id }`).
2. Nested `participants` → `MeetingParticipantBridge` with parent = `MeetingModel`.
3. Framework uses Meeting association `participants` (`hasMany` / `as: "participants"`).
4. Nested `member` → `MemberBridge` with parent = `MeetingParticipantModel`.

## 6) Seed

No participant seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — base + customer SDL/types include participant enums + `_MeetingParticipant` + nested `participants` |
| `cpanel/` | Deferred — no supervisor participant surface |

Verification scripts (backend): `yarn generate-types`, `yarn type-check`.

## 8) Agreed type → permission matrix (product contract; helper not shipped)

Explicit product decision (replaces a `can_vote` boolean): permissions are **fixed per `type` in code** when the helper ships. Not stored as JSONB on the row. Not implemented as a module in this change set.

| Permission | CHAIRPERSON | MEMBER | VIEWER |
|---|---|---|---|
| view meeting | yes | yes | yes |
| vote | yes | yes | no |
| request talking | yes | yes | no |
| manage live vote / queue | yes | no | no |
| start / complete meeting | yes | no | no |

Notes:

- `Meeting.chairperson_id` remains the meeting’s chairperson pointer; the roster row for that person should use `type = CHAIRPERSON` when writes ship (consistency rule for requesters later).
- `Member` has **no** org-wide role/permission fields for meeting session actions (directory person only).

## 9) Explicit non-goals confirmed in this work

| Topic | Decision |
|---|---|
| Surrogate join `id` | Not needed; keep composite PK |
| `belongsToMany` Meeting ↔ Member | Removed; use `hasMany` join only |
| Root participant queries | Declined |
| `_Member` → meetings / meetingParticipants GQL | Not needed yet (admin path is meeting → roster → member) |

## 10) Failure modes (read path)

Nested participants inherit the parent meeting read gate:

| Surface | Condition | Behavior |
|---|---|---|
| `meeting` root | no `context.customer` | `NOT_PERMIT` |
| `meeting` root | customer has no organization | `404` |
| `meeting(id)` | missing / other-org id | framework empty → `404` |
| nested `participants` | meeting not loaded / not permitted | not reached |

No separate root participant failure modes (no root queries).

## 11) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/MeetingParticipant.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Meeting.ts` | `hasMany` `participants` + mixins | §3.5 |
| `backend/src/app/orm/models/Member.ts` | `hasMany` MeetingParticipant + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | participant enums AR | §3.3 |
| `backend/src/resources/trans/en/general.ts` | participant enums EN | §3.3 |
| `backend/src/app/gql/definitions/base.graphql` | participant GQL enum wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_MeetingParticipant` + `_Meeting.participants` | §4 |
| `backend/src/app/gql/bridges/customer/MeetingParticipantBridge.ts` | nested bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | `GetOneParent` includes join model | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register bridge (no new roots) | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enums); no participant roots | §7 |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry (gitignored) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/invariants/backend.md` (B15)
- `.cursor/rules/meeting-participant-roster.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
