# Vote Domain Contract (Current)

## 1) Scope

Current Ejtmaa vote surface:

- ORM persistence for durable ballot rows linked to a `Decision` (and denormalized `meeting_id` for report queries),
- customer GraphQL **nested** read under `_Decision.votes`,
- nested `member` on each vote,
- website GQL mirrors for that nested surface.

Out of scope (not shipped):

- root queries `votes` / `vote(...)`,
- GraphQL `meeting` / `decision` on `_Vote` (parent path supplies them: meeting → decisions → votes),
- FK scalars on `_Vote` (`decision_id`, `member_id`, `meeting_id`),
- surrogate vote `id` (composite PK kept),
- vote write requesters / mutations,
- supervisor Vote GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for votes,
- live in-flight vote cursor (socket) — durable casts only in this model.

## 2) Domain purpose

`Vote` is a **non-actor** durable cast: one member’s ballot on one decision.

- Identity: composite PK `(decision_id, member_id)` — one vote per member per decision.
- `meeting_id` is denormalized for meeting-scoped report queries without joining through Decision every time.
- GraphQL nest path: `_Meeting.decisions` → `_Decision.votes` → `_Vote.member`.
- Explicit product decision: expose **`member` only** among relations on `_Vote`; do **not** nest `meeting` on `_Vote` while the nest path already provides meeting + decision.

## 3) ORM model

File: `backend/src/app/orm/models/Vote.ts`

Classification: **non-actor** (`Model<Attrs, Attrs>` — no surrogate `id`, no `Ability`, no `can()`).

Persistence names:

- `modelName`: `vote` (default `hasMany` from Decision → `votes` — **no** `as`)
- `tableName`: `votes`

### 3.1 Attrs layout

- `//relations` — `decision_id`, `member_id`, `meeting_id`
- `//info` — `value`, `cast_at`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `decision_id` | UUID | no | PK part; FK → Decision.id |
| `member_id` | UUID | no | PK part; FK → Member.id |
| `meeting_id` | UUID | no | FK → Meeting.id (denormalized) |
| `value` | STRING(191) | no | enum `voteValue`: `YES` \| `NO` |
| `cast_at` | DATE | no | when cast |

Exported TS type: `VoteValue`.

### 3.3 Enums (localized)

| Enum key | Values |
|---|---|
| `voteValue` | `YES`, `NO` |

AR/EN: `backend/src/resources/trans/ar/general.ts`, `en/general.ts`.

### 3.4 Indexes

- `votes_meeting_id`
- `votes_member_id`
- (composite PK covers uniqueness per decision+member)

### 3.5 Relations

`Vote.boot()`:

- `belongsTo(Decision)` on `decision_id`
- `belongsTo(Member)` on `member_id`
- `belongsTo(Meeting)` on `meeting_id`

`Decision.boot()`:

```ts
this.hasMany(Vote(), {
    sourceKey: "id",
    foreignKey: "decision_id"
});
```

No `as`. Mixins on Decision: `votes` / `getVotes` / `createVote` / ….

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_VoteValue` / `_VoteValueValue`
- `backend/src/app/gql/definitions/customer.graphql` — `_Vote` + `_Decision.votes`

### Type `_Vote`

Implements `_Timestamps` & `_Pagination`.

Info: `value`, `cast_at`.

Relations: `member: _Member`.

Timestamps / pagination: `created_at`, `updated_at`, `total_count`.

**Not exposed:** `id`, `decision_id`, `member_id`, `meeting_id`, `decision`, `meeting`.

### Nested under `_Decision`

```graphql
votes: [_Vote]
```

Cardinality: votes per decision ≤ roster size (B15 OK under meeting nest chain).

### Root queries

**None.**

### Bridge: `VoteBridge`

File: `backend/src/app/gql/bridges/customer/VoteBridge.ts`

- Extends `CustomerBridgeBase`
- `ident = "votes"`
- `typeIdent = "_Vote"`
- `ormModel = VoteModel`
- `GetManyParent = DecisionModel`
- `GetOneParent = VoteModel`

### Nested parent typing

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Decision.votes` | `VoteBridge` | `GetManyParent = DecisionModel` |
| `_Vote.member` | `MemberBridge` | `GetOneParent` includes `VoteModel` |

### Registered bridges

`CustomerSchema.registeredBridges` includes `VoteBridge`.

## 5) Read flow (nested)

```text
meeting(id) { decisions { votes { value cast_at member { id name } } } }
```

1. `MeetingBridge` (org-scoped root)
2. `DecisionBridge` with parent Meeting
3. `VoteBridge` with parent Decision (`votes` association)
4. `MemberBridge` with parent Vote

## 6) Seed

No vote seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — base + customer include `_VoteValue`, `_Vote`, `_Decision.votes` |
| `cpanel/` | Deferred |

Verification: `yarn generate-types`, `yarn type-check`; copy base + customer to `website/src/types/gql/**`.

## 8) Failure modes (read path)

Nested votes inherit meeting/decision read gates; no separate root failure modes.

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Vote.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Decision.ts` | `hasMany` Vote + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | `voteValue` AR | §3.3 |
| `backend/src/resources/trans/en/general.ts` | `voteValue` EN | §3.3 |
| `backend/src/app/gql/definitions/base.graphql` | `_VoteValue` | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_Vote` + `_Decision.votes` | §4 |
| `backend/src/app/gql/bridges/customer/VoteBridge.ts` | nested bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | `GetOneParent` + `VoteModel` | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register bridge | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enum); no vote roots | §7 |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry `Vote` (gitignored) | excluded |

## Related

- `docs/platforms/backend/contracts/decision-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `.cursor/rules/vote-decision-child.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
