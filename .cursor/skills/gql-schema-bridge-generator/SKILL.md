---
name: gql-schema-bridge-generator
description: Generates and updates Ejtmaa-compatible GraphQL schemas and bridges with strict security, mapping, pagination, and depth guardrails. Use when adding or modifying gql schemas, role bridge bases, entity bridges, me bridges, unions, and extras.
---

# GQL Schema Bridge Generator

## When to Use

- Creating a new role schema or extending an existing schema.
- Creating a new base bridge or entity bridge.
- Adding root queries, filters, extras, or union routing.
- Fixing complex GQL issues (N+1, pagination drift, auth leakage, depth/perf regressions).

## Inputs

Required:

- target schema role
- target bridge or bridges
- requested query or field behavior
- principal scope requirements
- pagination and sorting requirements

Optional but recommended:

- expected GraphQL args shape
- union or polymorphic discriminator mapping
- extras behavior expectations
- performance sensitivity notes

## Required References

- `docs/platforms/backend/patterns/gql-constitution-map.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `docs/platforms/backend/patterns/gql-schema-bridge-authoring-standard.md`
- `docs/platforms/backend/patterns/gql-source-corpus.md`
- `docs/platforms/backend/patterns/gql-pattern-atlas.md`
- `docs/platforms/backend/patterns/gql-complex-problems-cookbook.md`
- `docs/platforms/backend/patterns/gql-anti-patterns-blacklist.md`
- `docs/platforms/backend/patterns/gql-refusal-matrix.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/invariants/backend.md`

## Instructions

```text
Task Progress:
- [ ] Step 1: Classify request type (schema/base/entity/union/fix)
- [ ] Step 2: Domain mapping (canonical vs compatible vs drift)
- [ ] Step 3: Define context/principal and bridge query contract
- [ ] Step 4: Add security/depth/perf/query-safety guardrails
- [ ] Step 5: Validate framework + refusal matrix + scoring rubric + mirror sync
```

## Templates

### New Schema Template

1. Define `Context` with principal, pagination controls, `lang`, `by`.
2. Register required bridges in `registeredBridges`.
3. Keep resolvers thin: `Bridge.AsRoot(...).prepare...`.

### New Base Bridge Template

1. Define `schemaIdent`, `requiredOrmAttrs`, `maxLevel`.
2. Implement `withListable`, `withReplacements`, `getRootOrmParent`, `getOrmFindOptions`.
3. Add `deepInclude` and `ignoreExtra` defaults.
4. Define supported parent modes explicitly (`me/public/self` + aliases).
5. **Mandatory**: implement `loadAttr` override (MultiLang JSONB → string, enum → select), `trans`, `toEnum`, `toManyEnums`, `willPrepare`.
   - Reference: `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`.
   - Without this, any JSONB field declared as `String` in SDL will abort entire queries.

### New Entity Bridge Template

1. Define `ident`, `typeIdent`, `ormModel`.
2. Map args to typed parent payload.
3. Ensure any sort/filter/extra attr is loaded.
4. If root-one is actor-bound and parent is `STATIC`, enforce ownership scope in `where`.

### Union-Ready Template

1. Define union include idents and discriminators.
2. Keep deterministic union routing.
3. Verify discriminator attrs are always selected.

### Secure Me Bridge Template

1. Guard actor in `willPrepare` and/or `getRootOrmParent`.
2. Resolve model by context principal id.
3. Keep one-only behavior deterministic.

## Refusal Conditions

Refuse finalization until clarified if:

- principal contract is missing for scoped queries,
- root-many mode/order is undefined,
- ORM attr loading strategy is incomplete,
- requested dynamic SQL filter requires unsafe interpolation,
- framework contract change is ambiguous (provider/schema/bridge boot invariants),
- requested style conflicts with Ejtmaa constitution.

## Output Requirements

- Provide changed files list with short rationale.
- Explicitly state security, mapping, pagination, and depth checks performed.
- State which cookbook playbook(s) were used for complex fixes.
- Explicitly state enum output wrapper checks (`_X` output type vs `_XValue` filter enum).
- Explicitly state backend->frontend mirror sync status for:
  - `website/src/types/gql/definitions/*.graphql` and `website/src/types/gql/gql-types/*.ts` (customer) when `website/` exists
  - `cpanel/src/types/gql/definitions/*.graphql` and `cpanel/src/types/gql/gql-types/*.ts` (supervisor) only when `cpanel/` exists; if absent, state deferred and do not create mirrors
- Explicitly state role bridge base contract compliance (loadAttr + trans + toEnum + willPrepare).

## Quality Gates

- compatibility gate: Ejtmaa style and ownership contracts remain intact
- security gate: scoped reads keep principal protection in the correct owner
- mapping gate: no sort/filter/extra/union path depends on unloaded ORM attrs
- pagination gate: root-many behavior stays bounded and deterministic
- depth/perf gate: finite depth and include controls remain active

## Failure Handling

If partially blocked:

1. return the completed subset
2. list the unresolved contract gaps
3. provide the minimal follow-up inputs needed
4. avoid speculative architecture decisions

## Ejtmaa Bridge Addendum (Strict)

Current Ejtmaa GQL surfaces:

- Customer: `me`, `notifications`, `organization`, `members`, `member(id)`, `messageTemplates`, `messageTemplate(id)`, `meetings`, `meeting(id)` (+ nested `participants`, `agendaItems`, `decisions`, `talkRecords`), `plans`, `plan(id)`
- Supervisor: `me`, `notifications`, `customers`, `customer`, `customerStats`, `organizations`, `organization`

Reference bridges:

- `backend/src/app/gql/bridges/customer/MeBridge.ts`
- `backend/src/app/gql/bridges/customer/NotificationBridge.ts`
- `backend/src/app/gql/bridges/customer/OrganizationBridge.ts`
- `backend/src/app/gql/bridges/customer/MemberBridge.ts`
- `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts`
- `backend/src/app/gql/bridges/customer/MeetingBridge.ts`
- `backend/src/app/gql/bridges/customer/MeetingParticipantBridge.ts`
- `backend/src/app/gql/bridges/customer/AgendaItemBridge.ts`
- `backend/src/app/gql/bridges/customer/DecisionBridge.ts`
- `backend/src/app/gql/bridges/customer/VoteBridge.ts`
- `backend/src/app/gql/bridges/customer/TalkRecordBridge.ts`
- `backend/src/app/gql/bridges/customer/PlanBridge.ts`
- `backend/src/app/gql/bridges/supervisor/MeBridge.ts`
- `backend/src/app/gql/bridges/supervisor/NotificationBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`
- `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts`
- `backend/src/app/gql/bridges/supervisor/OrganizationBridge.ts`

Rules:

- `CustomerStatsBridge.loadExtra` serves `total_count`.
- `CustomerBridge` owns supervisor list/detail filter mapping.
- Customer `organization` root-one: `prepareOneGQLModel({ me: true })` when bridge `ident` matches `Customer.hasOne` association key; do not invent `as` / `getRootOrmParent` overrides.
- Customer `members` / `member(id)`: `{ me: true }` resolves root parent to the customer's Organization (Member belongs to Organization, not Customer); no supervisor Member surface yet.
- Customer `messageTemplates` / `messageTemplate(id)`: same `{ me: true }` → Organization parent pattern as members; inverse `_MessageTemplate.organization` requires `OrganizationBridge` `GetOneParent` to include `MessageTemplateModel`.
- Customer `meetings` / `meeting(id)`: same org-owned base; nested `chairperson` / `whatsappTemplate` / `emailTemplate` require `MemberBridge` / `MessageTemplateBridge` `GetOneParent` to include `MeetingModel`; inverse `_Meeting.organization` requires `OrganizationBridge` `GetOneParent` to include `MeetingModel`.
- Customer `_Meeting.participants: [_MeetingParticipant]` (nested only; no root): join-row fields (`type`, `notified`, `delivery_status`, `attended_at`, `left_at`) + `member`; no FK scalars on `_MeetingParticipant`. ORM: `Meeting.hasMany(MeetingParticipant, { as: "participants" })` only (no `belongsToMany`). `MeetingParticipantBridge.ident = "participants"`; `MemberBridge.GetOneParent` includes `MeetingParticipantModel`. Contract: `meeting-participant-domain.md`.
- Customer `_Meeting.agendaItems: [_AgendaItem]` (nested only; no root): `id`, `sort_order`, `subject`; no `meeting_id` scalar. ORM: `Meeting.hasMany(AgendaItem)` (default association; no `as`). `AgendaItemBridge.ident = "agendaItems"`.
- Customer `_Meeting.decisions: [_Decision]` (nested only; no root): `id`, `sort_order`, `subject`, `phase`, `status`, `voting_type`; no `meeting_id` scalar. ORM: `Meeting.hasMany(Decision)` (default association; no `as`). `DecisionBridge.ident = "decisions"`. Base enums: `_DecisionPhase` / `_DecisionStatus` / `_DecisionVotingType`.
- Customer `_Decision.votes: [_Vote]` (nested only; no root): `value`, `cast_at`, `member`; no FK scalars. ORM: `Decision.hasMany(Vote)` (default; no `as`). `VoteBridge.ident = "votes"`; `MemberBridge.GetOneParent` includes `VoteModel`. Base enum: `_VoteValue`.
- Customer `_Meeting.talkRecords: [_TalkRecord]` (nested only; no root): `id`, `sort_order`, `status`, `started_at`, `ended_at`, `member`; no FK scalars / no `decision`. ORM: `Meeting.hasMany(TalkRecord)` (default; no `as`). `TalkRecordBridge.ident = "talkRecords"`; `MemberBridge.GetOneParent` includes `TalkRecordModel`. Base enum: `_TalkRecordStatus`.
- Customer `plans` / `plan(id)`: platform catalog (no owner FK). Parent `{ public: true }` → `STATIC`. `PlanBridge` extends `CustomerBridgeBase` (not org-owned). Root policy: `status = ACTIVE`, order `sort_order` then `id`. MultiLang `name`/`description` → localized `String`. Base enums: `_PlanStatus`, `_PlanBillingPeriod`. No supervisor Plan surface yet.
- Do **not** add `_Member.meetingParticipants` / `_Member.meetings` unless product requests member-history UX (B15 risk over time).
- When adding nested SDL `belongsTo` (example `_Member.organization`), update the **target** bridge `GetOneParent` to include the **source** model (`OrganizationBridge`: `MemberModel | …`). Never skip this. See `gql-root-parent-payload-contract.mdc` §5 and `member-domain.md`.
- Do not nest high-cardinality `hasMany` under parent types when expected count may exceed 100 (B15); use root list instead. Meeting roster under `_Meeting` is an allowed nest for expected board size.
- Supervisor `OrganizationBridge` owns organization list/detail filters; nested `_Customer.organization` needs the bridge registered.
- Role bridge bases: `CustomerBridgeBase`, `SupervisorBridgeBase`.
- Org-owned customer children (`Member`, `MessageTemplate`, `Meeting`, …): extend `CustomerOrganizationOwnedBridgeBase` (`me` → customer's Organization). Do not copy that `getRootOrmParent` into each entity bridge.
- Keep resolvers thin; ORM policy lives in bridges.
- Sync mirrors after SDL changes: `website/` (`customer`) always when present.
- `cpanel/` (`supervisor`) sync only when the `cpanel/` platform checkout exists; currently deferred (folder temporarily removed) — do not invent `cpanel/src/types/gql/**` mirrors.
- Organization contract detail: `docs/platforms/backend/contracts/organization-domain.md`.
