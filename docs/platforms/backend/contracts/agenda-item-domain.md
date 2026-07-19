# AgendaItem Domain Contract (Current)

## 1) Scope

Current Ejtmaa agenda surface:

- ORM persistence for meeting agenda lines (durable SQL, prepared before/around the meeting),
- customer GraphQL **nested** read under `_Meeting.agendaItems`,
- website GQL mirrors for that nested surface.

Out of scope (not shipped):

- root queries `agendaItems` / `agendaItem(id)`,
- GraphQL inverse `_AgendaItem.meeting`,
- root agenda write requesters (writes ship as `MeetingRequester` agenda subs — see `meeting-domain.md` §9),
- supervisor AgendaItem GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for agenda items,
- Yjs / LiveKit ownership of agenda (explicitly not used — SQL is source of truth).

## 2) Domain purpose

`AgendaItem` is a **non-actor** ordered line item belonging to a `Meeting`.

- Agenda is authored and stored in SQL so it exists **before** the live session and can feed reports later.
- Not a CRDT document and not LiveKit state.
- Tenant isolation is inherited via `Meeting.organization_id`.

## 3) ORM model

File: `backend/src/app/orm/models/AgendaItem.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `agendaItem` (Sequelize default `hasMany` association name → `agendaItems` — **no** `as` alias)
- `tableName`: `agenda_items`

### 3.1 Attrs layout

- `//relations` — `meeting_id`
- `//info` — `sort_order`, `subject`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID PK | no | default `UUIDV4` |
| `meeting_id` | UUID | no | FK → Meeting.id |
| `sort_order` | INTEGER | no | display order within meeting |
| `subject` | STRING(191) | no | agenda line text |

### 3.3 Indexes

- `agenda_items_meeting_id` — list by meeting

### 3.4 Relations

`AgendaItem.boot()`:

- `belongsTo(Meeting)` on `meeting_id`

`Meeting.boot()`:

```ts
this.hasMany(AgendaItem(), {
    sourceKey: "id",
    foreignKey: "meeting_id"
});
```

No `as` — association key is the default plural `agendaItems`.

Mixin declare block on Meeting: `agendaItems` / `getAgendaItems` / `createAgendaItem` / … (association PK type `string` for UUID agenda id).

## 4) Customer GraphQL surface

SDL: `backend/src/app/gql/definitions/customer.graphql`

### Type `_AgendaItem`

Implements `_Timestamps` & `_Pagination`.

Info: `id`, `sort_order`, `subject`.

Timestamps: `created_at`, `updated_at`.

Pagination: `total_count`.

**Not exposed:** `meeting_id` scalar; no nested `meeting` relation yet.

### Nested under `_Meeting`

```graphql
agendaItems: [_AgendaItem]
```

Cardinality gate (B15): agenda lines for a board meeting are expected well under 100 → nested list allowed.

### Root queries

**None.**

### Bridge: `AgendaItemBridge`

File: `backend/src/app/gql/bridges/customer/AgendaItemBridge.ts`

- Extends `CustomerBridgeBase` (not org-owned root base — no root `me` path)
- `ident = "agendaItem"` (must match ORM `modelName`; association key `agendaItems` is the SDL/include field)
- `typeIdent = "_AgendaItem"`
- `ormModel = AgendaItemModel`
- `GetManyParent = MeetingModel`
- `GetOneParent = AgendaItemModel`
- No `getRootOrmParent` / `getOrmFindOptions` override

### Nested parent typing

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Meeting.agendaItems` | `AgendaItemBridge` | `GetManyParent = MeetingModel` |

### Registered bridges

`CustomerSchema.registeredBridges` includes `AgendaItemBridge`. No new Query resolvers.

## 5) Read flow (nested)

### `meeting(id) { agendaItems { id sort_order subject } }`

1. Root `MeetingBridge` loads the meeting (org-scoped via `{ me: true, id }`).
2. Nested `agendaItems` → `AgendaItemBridge` with parent = `MeetingModel`.
3. Framework uses Meeting association `agendaItems` (`hasMany`, no `as`).

## 6) Seed

No agenda seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — customer SDL/types include `_AgendaItem` + nested `agendaItems` |
| `cpanel/` | Deferred — no supervisor agenda surface |

Verification scripts (backend): `yarn generate-types`, `yarn type-check`; then command-based copy of customer SDL/types to `website/src/types/gql/**`.

## 8) Naming invariant (learned)

Prefer a **basic** Sequelize `modelName` whose default plural matches the desired GQL field / bridge `ident` (here `agendaItem` → `agendaItems`) so Meeting does **not** need `as: "…"`.  
Contrast: `MeetingParticipant` uses `as: "participants"` because `modelName` is `meetingParticipant` (default would be `meetingParticipants`).

## 9) Failure modes (read path)

Nested agenda inherits the parent meeting read gate:

| Surface | Condition | Behavior |
|---|---|---|
| `meeting` root | no `context.customer` | `NOT_PERMIT` |
| `meeting` root | customer has no organization | `404` |
| `meeting(id)` | missing / other-org id | framework empty → `404` |
| nested `agendaItems` | meeting not loaded / not permitted | not reached |

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/AgendaItem.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Meeting.ts` | `hasMany` AgendaItem + mixins | §3.4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_AgendaItem` + `_Meeting.agendaItems` | §4 |
| `backend/src/app/gql/bridges/customer/AgendaItemBridge.ts` | nested bridge | §4–§5 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register bridge (no new roots) | §4 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry key `AgendaItem` (gitignored) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/invariants/backend.md` (B15)
- `.cursor/rules/agenda-item-meeting-child.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
