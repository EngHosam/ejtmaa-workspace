# Member Domain Contract (Current)

## 1) Scope

Current Ejtmaa member surface:

- ORM persistence for organization members (directory persons),
- customer GraphQL read of members for the authenticated customer's organization,
- website GQL mirrors for that customer surface.

Also shipped:

- demo members via `SeedPatch.update("test_seed")` → `seedDemoMembers` (mechanism only; payload in code),
- optional server-side list filter `_MemberFilter.search` on root `members` (name / email / mobile `iLike`),
- website authenticated members directory UI (list + search + load-more) — see `docs/platforms/website/flow-customer-members.md`,
- website Member write path via `MemberRequester` (`read` | `create` | `update` | `delete`) + `Customer.Ability.MEMBER` — see §10.

Out of scope (not shipped):

- supervisor Member GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- GraphQL `can*` ability fields on `_Me` / `_Member` for UI gating,
- soft-delete / cascade delete of roster or chairperson rows,
- role / permissions fields on Member (session permissions live on `MeetingParticipant.type` — see `meeting-participant-domain.md` §8),
- nested `_Organization.members` (high cardinality — root list only),
- nested `_Member.meetingParticipants` / `_Member.meetings` (not needed yet; admin path is meeting → roster → member),
- additional filter fields beyond `search` (no status/scope/sort args on `_MemberFilter`).

## 2) Domain purpose

`Member` is a **non-actor** person record inside an `Organization`.

- Access model is link-oriented: stable UUID `id` plus separate unique `access_token`.
- No `User` row; Customer remains the login actor and is **not** a Member row.
- Tenant boundary is `organization_id` (not `customer_id` directly).

## 3) ORM model

File: `backend/src/app/orm/models/Member.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id" | "avatar_url" | "access_token">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `member`
- `tableName`: `members`

### 3.1 Attrs layout

- `//relations` — `organization_id`
- `//info` — profile + `access_token`
- `//virtual` — `avatar_url`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID PK | no | default `UUIDV4` — opaque identity (no separate `public_id`) |
| `organization_id` | BIGINT | no | FK → `organizations.id`, real constraint |
| `name` | STRING(191) | no | |
| `email` | STRING(191) | yes | email + lowercase validators when set |
| `mobile` | STRING(191) | yes | |
| `avatar_file` | STRING(191) | yes | |
| `access_token` | UUID | no | default `UUIDV4`; unique index |
| `avatar_url` | VIRTUAL | — | `getImagePath(avatar_file)` |

### 3.3 Indexes

- `members_organization_id` — list by org
- `members_unique_access_token` — UNIQUE(`access_token`)

### 3.4 Relations

`Member.boot()`:

- `belongsTo(Organization)` on `organization_id` (real FK)
- `hasMany(MeetingParticipant)` on `member_id` (ORM inverse for roster; GQL nest under Member not exposed)

`Organization.boot()` inverse:

- `hasMany(Member)` on `organization_id` (real FK)
- mixins: `getMembers` / `createMember` / … (association PK type `string` for UUID member id)

No direct `Customer` ↔ `Member` association.

Do **not** add `belongsToMany(Meeting)` on Member for roster traversal.

## 4) Customer GraphQL surface

SDL: `backend/src/app/gql/definitions/customer.graphql`

### Type `_Member`

Implements `_Timestamps & _Pagination`.

Fields:

- info: `id`, `name`, `email`, `mobile`, `avatar_file`, `avatar_url`, `access_token`
- timestamps: `created_at`, `updated_at`
- relations: `organization: _Organization` (`belongsTo`, expected count 1 — cardinality-safe)
- pagination: `total_count`

Not exposed:

- `organization_id` column (use `organization` relation)
- nested `_Organization.members` (B15 high-cardinality gate; use root list)

### Filter input

```graphql
input _MemberFilter {
    search: String
}
```

- Optional. Omitted / null / whitespace-only → no text predicate (org-scoped list only).
- Non-empty after trim → case-insensitive substring match (`Op.iLike` with `%…%`) on **`name` OR `email` OR `mobile`**.

### Root queries

- `members(filter: _MemberFilter): [_Member]`
- `member(id: ID!): _Member`

Resolvers (`CustomerSchema`):

```ts
prepareManyGQLModels({ me: true, filter: filter || undefined })
prepareOneGQLModel({ me: true, id })
```

### Bridge: `MemberBridge`

File: `backend/src/app/gql/bridges/customer/MemberBridge.ts`

- Extends `CustomerOrganizationOwnedBridgeBase` (shared `me` → Organization resolve)
- `ident = "member"`, `typeIdent = "_Member"`, `ormModel = MemberModel`
- `MemberFilter` / `GetManyParent = OrganizationOwnedMeParent & { filter?: Nullable<MemberFilter> }`
- `GetOneParent = MemberModel | MeetingModel | MeetingParticipantModel | VoteModel | TalkRecordModel | { me: true; id: string }`
  - `MeetingModel` for `_Meeting.chairperson`
  - `MeetingParticipantModel` for `_MeetingParticipant.member`
  - `VoteModel` for `_Vote.member`
  - `TalkRecordModel` for `_TalkRecord.member`
- Does **not** override `getRootOrmParent` (org-owned base)
- **Does** override `getOrmFindOptions` for root `prepareType === "many"`:
  - Always applies `withListable()` + `withReplacements()` + `order: [["updated_at", "desc"]]`
  - When `filter.search` trims non-empty, adds `where: { [Op.and]: [{ [Op.or]: [name|email|mobile iLike] }] }`
  - Nested / one prepares fall through to `super.getOrmFindOptions`

Shared base: `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` — see `message-template-domain.md` §4.  
Parent-payload discipline: `.cursor/rules/gql-root-parent-payload-contract.mdc` (§3 — filter mapping is entity-owned policy).

### Inverse relation parent typing (mandatory)

When `_Member.organization` is selected, the framework prepares **`OrganizationBridge`** with parent = `MemberModel`.

Therefore `OrganizationBridge` must declare (also includes other inverse parents as they ship):

```ts
export type GetOneParent = MemberModel | MessageTemplateModel | MeetingModel | { me: true };
```

`MemberBridge.GetOneParent` also includes `MeetingModel` / `MeetingParticipantModel` / `VoteModel` / `TalkRecordModel` for nested meeting surfaces.

Do not put `OrganizationModel` on `MemberBridge.GetOneParent` for the organization inverse; that bridge prepares Organization, not Member.

### Registered bridges

`CustomerSchema.registeredBridges` includes `MemberBridge` (required for nested `organization` resolution).

## 5) Read flow (root)

### `members`

1. `MemberBridge.AsRoot` → `prepareManyGQLModels({ me: true, filter })`
2. `getRootOrmParent` → customer's Organization
3. `getOrmFindOptions` (many): listable + replacements + `updated_at DESC` + optional search `where`
4. `organization.getMembers(...)` — tenant scope remains the association; filter only narrows within the org

### `member(id)`

1. `prepareOneGQLModel({ me: true, id })`
2. Same org resolve as above
3. Base one options: `where: { id }`
4. Scoped to that organization association
5. `_MemberFilter` does **not** apply to the singular root

## 6) Seed mechanism

`SeedPatch.seedDemoMembers` (from `test_seed` after customers + orgs):

- early-return when `Member.count() > 0`
- seeds members for an explicit subset of organizations looked up by `subdomain` via `seedMembersForSubdomain(...)` — **not** a generic “plan/plans” structure (that name is the product ORM model `Plan`)
- member display names come from a curated Saudi name list in `SeedPatch.ts` (not faker)
- rows created with `organization.createMember(...)`; emails/mobiles/counts stay in code only

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — `customer.graphql` + `gql-types/customer.ts` synced |
| `cpanel/` | Deferred — platform checkout absent; no supervisor Member surface anyway |

Backend verification: `yarn generate-types`, `yarn type-check`.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `members` / `member` | no `context.customer` | `NOT_PERMIT` |
| `members` / `member` | customer has no organization | `404` |
| `member(id)` | missing / other-org id | framework empty → `404` |
| `members` + filter | search matches nothing in org | empty list (HTTP/GQL success; not 404) |

## 9) Write surface (`MemberRequester`)

HTTP: existing customer form controller → `API.FORMS.CUSTOMER.R("member")(sub)` → `/forms/customer/requester/member/:sub`. No new controller.

### 9.1 Ability — `Customer.Ability.MEMBER`

File: `backend/src/app/orm/models/Customer.ts`

```ts
MEMBER: {
  sub: "create" | "read" | "update" | "delete"
  member?: ModelRef<MemberModel>
}
```

| Sub | Guard |
|---|---|
| `create` | Customer has an organization; else `ACTION_NOT_ALLOWED` |
| `read` / `update` / `delete` | Resolve `member`; must belong to that organization (`404` missing, `NOT_PERMIT` other org) |
| `delete` (extra) | Block if any `MeetingParticipant` with `member_id` → `CANNOT_DELETE_MEMBER_IN_MEETING_ROSTER`; or any `Meeting` with `chairperson_id` → `CANNOT_DELETE_MEMBER_IS_CHAIRPERSON`. **No cascade.** |

### 9.2 Joi — `isCustomerOwnedMember`

File: `backend/src/app/validation/joi_rules.ts`

- `Member().Opt` + external: requires fact `customer`, loads customer org, rejects when `member.organization_id` ≠ org id (`NOT_PERMIT`).
- Used on `read` / `update` / `delete` for the `member` ref.

Field rules (requester-local helpers):

| Field | Rule |
|---|---|
| `name` | `joi.string().trim().min(2)` — keep internal spaces |
| `email` | optional; trim + lowercase + email; `""` / null → null; machine whitespace cleanup allowed |
| `mobile` | optional `isMobile`; `""` / null → null; machine whitespace cleanup allowed |
| `avatar_file` | optional string filename; `""` / null → null on persist |

### 9.3 Requester subs

File: `backend/src/app/orchestrator/requesters/MemberRequester.ts` (`@requester("member")`, platform `website`, actor `customer`).

`MemberRef = string | SelectOption | MemberModel`.

| Sub | Behavior |
|---|---|
| `read` | Validate facts + owned member → `can MEMBER read` → values `{ member, name, email, mobile, avatar_file }` |
| `create` | Validate fields → `can MEMBER create` → `organization.createMember(...)` → `SUCCESS_CREATE` |
| `update` | Owned member + fields; apply `email` / `mobile` / `avatar_file` only when `!== undefined` (omit must not wipe) → `can MEMBER update` → `member.update` → `SUCCESS_UPDATE` |
| `delete` | Owned member → `can MEMBER delete` (roster/chairperson block) → `member.destroy` → `SUCCESS_DELETE` |

`outPropsType`: `Modify` only for `member → MemberModel` on read/update/delete; create = `CreateProps & { customer }`.

### 9.4 Maps + messages

| Map | Entry |
|---|---|
| `backend/requesters.website.ts` | `customer.member: "read" \| "create" \| "update" \| "delete"` |
| `website/src/types/requesters/requesters.website.ts` | Same union (W18) |

Messages (`backend/src/resources/trans/{ar,en}/messages.ts`):

- `SUCCESS_CREATE` / `SUCCESS_UPDATE` / `SUCCESS_DELETE`
- `CANNOT_DELETE_MEMBER_IN_MEETING_ROSTER`
- `CANNOT_DELETE_MEMBER_IS_CHAIRPERSON`

### 9.5 Failure modes (write)

| Condition | Result |
|---|---|
| No org on create | `ACTION_NOT_ALLOWED` |
| Member missing | `404` |
| Other-org member | `NOT_PERMIT` |
| Delete while on roster | `CANNOT_DELETE_MEMBER_IN_MEETING_ROSTER` |
| Delete while chairperson | `CANNOT_DELETE_MEMBER_IS_CHAIRPERSON` |
| Validation errors | field / firewall errors via `throwIfNotValid` |

Verify: `yarn type-check` in `backend/`.

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Member.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/MeetingParticipant.ts` | ORM roster inverse (GQL nest deferred) | §3.4; delete guard §9.1 |
| `backend/src/app/orm/models/Organization.ts` | `hasMany Member` + mixins | §3.4 |
| `backend/src/app/orm/models/Customer.ts` | `Ability.MEMBER` + `can("MEMBER")` | §9.1 |
| `backend/src/app/validation/joi_rules.ts` | `isCustomerOwnedMember` | §9.2 |
| `backend/src/app/orchestrator/requesters/MemberRequester.ts` | write requester | §9.3 |
| `backend/requesters.website.ts` | website map | §9.4 |
| `backend/src/resources/trans/ar/messages.ts` / `en/messages.ts` | delete + success keys | §9.4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_Member` + `_MemberFilter` + roots + inverse relation | §4 |
| `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | shared `me` → Organization | §4 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | org-owned bridge + search `getOrmFindOptions` | §4–§5 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `MemberModel` for inverse | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register + resolvers (`filter` parent) | §4 |
| `backend/src/app/orm/patches/SeedPatch.ts` | `seedDemoMembers` / `test_seed` | §6 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated (`_MemberFilter`) | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7; `flow-customer-members.md` |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7; `flow-customer-members.md` |
| `website/src/types/requesters/requesters.website.ts` | `customer.member` mirror | §9.4 |
| `backend/.types/models.ts` | Generated registry (gitignored) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/message-template-domain.md` (shared org-owned GQL base)
- `docs/platforms/backend/contracts/meeting-participant-domain.md` (roster join; session type permissions)
- `docs/platforms/backend/contracts/vote-domain.md` (`_Vote.member`)
- `docs/platforms/backend/contracts/talk-record-domain.md` (`_TalkRecord.member`)
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/website/flow-customer-members.md` (website directory + form UI)
- `docs/platforms/website/flow-form-foundation.md` (`Forms.CUSTOMER_MEMBER`)
- `docs/invariants/backend.md` (B15, B18, B23)
- `.cursor/rules/gql-root-parent-payload-contract.mdc` (explicit `filter` parent + inverse `GetOneParent`)
- `.cursor/rules/meeting-participant-roster.mdc`
- `.cursor/skills/backend-requester-governance/SKILL.md`
