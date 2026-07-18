# Decision Domain Contract (Current)

## 1) Scope

Current Ejtmaa decision surface:

- ORM persistence for meeting decisions (durable SQL; pre-start and during-meeting phases),
- customer GraphQL **nested** read under `_Meeting.decisions`,
- localized enums for phase / status / voting type,
- website GQL mirrors for that nested surface.

Out of scope (not shipped):

- root queries `decisions` / `decision(id)`,
- GraphQL inverse `_Decision.meeting`,
- nested votes / talk records under `_Decision` (models not shipped yet),
- decision write requesters / mutations,
- supervisor Decision GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for decisions,
- Yjs ownership of decisions (SQL is source of truth).

## 2) Domain purpose

`Decision` is a **non-actor** ordered decision item belonging to a `Meeting`.

- `phase = PRE_START` — created/edited before the meeting starts.
- `phase = DURING` — created or advanced during the live session.
- Status / voting-type fields support later vote rows and reports; votes themselves are a separate model (not in this change set).
- Tenant isolation is inherited via `Meeting.organization_id`.

## 3) ORM model

File: `backend/src/app/orm/models/Decision.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `decision` (default `hasMany` → `decisions` — **no** `as` alias)
- `tableName`: `decisions`

### 3.1 Attrs layout

- `//relations` — `meeting_id`
- `//info` — `sort_order`, `subject`, `phase`, `status`, `voting_type`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID PK | no | default `UUIDV4` |
| `meeting_id` | UUID | no | FK → Meeting.id |
| `sort_order` | INTEGER | no | display order |
| `subject` | STRING(191) | no | decision text |
| `phase` | STRING(191) | no | enum `decisionPhase` |
| `status` | STRING(191) | no | enum `decisionStatus`; default `NEW` |
| `voting_type` | STRING(191) | yes | enum `decisionVotingType`; null until voting opens |

Exported TS types: `DecisionPhase`, `DecisionStatus`, `DecisionVotingType`.

### 3.3 Enums (localized)

Under `backend/src/resources/trans/ar/general.ts` and `en/general.ts`:

| Enum key | Values |
|---|---|
| `decisionPhase` | `PRE_START`, `DURING` |
| `decisionStatus` | `NEW`, `UNDER_VOTING`, `ACCEPTED`, `REJECTED` |
| `decisionVotingType` | `LIVE`, `SECRET` |

### 3.4 Indexes

- `decisions_meeting_id`
- `decisions_meeting_id_phase`

### 3.5 Relations

`Decision.boot()`:

- `belongsTo(Meeting)` on `meeting_id`

`Meeting.boot()`:

```ts
this.hasMany(Decision(), {
    sourceKey: "id",
    foreignKey: "meeting_id"
});
```

No `as` — association key is default plural `decisions`.

Mixin declare block on Meeting: `decisions` / `getDecisions` / `createDecision` / … (PK type `string`).

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_DecisionPhase`, `_DecisionStatus`, `_DecisionVotingType` (+ Value enums)
- `backend/src/app/gql/definitions/customer.graphql` — `_Decision` + `_Meeting.decisions`

### Type `_Decision`

Implements `_Timestamps` & `_Pagination`.

Info: `id`, `sort_order`, `subject`, `phase`, `status`, `voting_type`.

Timestamps: `created_at`, `updated_at`.

Pagination: `total_count`.

**Not exposed:** `meeting_id` scalar; no nested `meeting` / votes / talk records yet.

### Nested under `_Meeting`

```graphql
decisions: [_Decision]
```

Cardinality gate (B15): expected board decision count well under 100 → nested list allowed.

### Root queries

**None.**

### Bridge: `DecisionBridge`

File: `backend/src/app/gql/bridges/customer/DecisionBridge.ts`

- Extends `CustomerBridgeBase`
- `ident = "decisions"` (matches Meeting default association)
- `typeIdent = "_Decision"`
- `ormModel = DecisionModel`
- `GetManyParent = MeetingModel`
- `GetOneParent = DecisionModel`
- No `getRootOrmParent` / `getOrmFindOptions` override
- Enum fields use role-base `loadAttr` → `getAsSelect` (ORM `enum` + `isTypedObject`)

### Nested parent typing

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Meeting.decisions` | `DecisionBridge` | `GetManyParent = MeetingModel` |

### Registered bridges

`CustomerSchema.registeredBridges` includes `DecisionBridge`. No new Query resolvers.

## 5) Read flow (nested)

### `meeting(id) { decisions { id phase status … } }`

1. Root `MeetingBridge` loads the meeting (org-scoped via `{ me: true, id }`).
2. Nested `decisions` → `DecisionBridge` with parent = `MeetingModel`.
3. Framework uses Meeting association `decisions` (`hasMany`, no `as`).

## 6) Seed

No decision seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — base + customer SDL/types include decision enums + `_Decision` + nested `decisions` |
| `cpanel/` | Deferred — no supervisor Decision surface |

Verification: `yarn generate-types`, `yarn type-check`; copy `base` + `customer` SDL/types to `website/src/types/gql/**`.

## 8) Naming invariant

Same as AgendaItem: basic `modelName: "decision"` → default plural `decisions` so Meeting needs no `as`. See `.cursor/rules/agenda-item-meeting-child.mdc` naming preference and `.cursor/rules/decision-meeting-child.mdc`.

## 9) Failure modes (read path)

Nested decisions inherit the parent meeting read gate:

| Surface | Condition | Behavior |
|---|---|---|
| `meeting` root | no `context.customer` | `NOT_PERMIT` |
| `meeting` root | customer has no organization | `404` |
| `meeting(id)` | missing / other-org id | framework empty → `404` |
| nested `decisions` | meeting not loaded / not permitted | not reached |

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Decision.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Meeting.ts` | `hasMany` Decision + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | decision enums AR | §3.3 |
| `backend/src/resources/trans/en/general.ts` | decision enums EN | §3.3 |
| `backend/src/app/gql/definitions/base.graphql` | decision GQL enum wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_Decision` + `_Meeting.decisions` | §4 |
| `backend/src/app/gql/bridges/customer/DecisionBridge.ts` | nested bridge | §4–§5 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register bridge (no new roots) | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enums); no decision roots | §7 |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry key `Decision` (gitignored) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/agenda-item-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/invariants/backend.md` (B15)
- `.cursor/rules/decision-meeting-child.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
