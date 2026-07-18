---
name: orm-model-generator
description: Generates and updates Ejtmaa ORM models with actor/non-actor contracts and ORM-to-GQL integration rules. Use when creating a new model, modifying model attrs/options/relations/can ability logic, or wiring model fields into GraphQL bridges and schemas.
---

# ORM Model Generator

## When to Use

- Use when creating a new ORM model in `backend/src/app/orm/models`.
- Use when modifying model `Attrs`, `attributes()`, `initOptions()`, `boot()`, or helper functions.
- Use when adding actor abilities (`Ability` + `can()`).
- Use when a model change must remain compatible with GQL bridges/schemas.

## Inputs

Required:

- target model name
- business purpose
- actor or non-actor classification
- required fields
- required relations

Optional but recommended:

- requester touchpoints
- bridge/schema touchpoints
- ability matrix for actor paths
- sorting/filtering requirements

## Required References

Read these before implementation:

- `docs/platforms/backend/patterns/orm-constitution-map.md`
- `docs/platforms/backend/patterns/orm-model-baseline-audit.md`
- `docs/platforms/backend/patterns/orm-model-authoring-standard.md`
- `docs/platforms/backend/patterns/orm-model-compliance-checklists.md`
- `docs/platforms/backend/patterns/orm-model-worked-examples.md`
- `docs/platforms/backend/patterns/models-owners-abilities-security.md`
- `docs/platforms/backend/patterns/graphql-and-bridges.md`

## Instructions

Use this checklist and keep exactly one step in progress:

```text
Task Progress:
- [ ] Step 1: Classify model path (actor/non-actor)
- [ ] Step 2: Domain/function mapping for model responsibilities
- [ ] Step 3: Design Attrs/options/relations and ability contract
- [ ] Step 4: Map ORM fields to GQL touchpoints
- [ ] Step 5: Validate with checklist and security gates
```

## Step 1: Classify Model Path

Choose **actor path** if one or more are true:

- model represents owner identity domain,
- model enforces permissioned business actions,
- model requires polymorphic owner semantics,
- requester/bridge needs `can(...)` decisions.

Else choose **non-actor path**.

## Step 2: Design Structure

Required model structure:

1. `type Attrs`.
2. `static attributes()`.
3. `static initOptions()`.
4. `static async boot()`.
5. Factory export: `export const X = () => ormModel(m => m.X) as typeof XModel;`.

Rules:

- Include virtual fields used by callers in `Attrs`.
- Keep indexes/scopes minimal and purposeful.
- Keep relations aligned with expected requester/bridge traversal paths.

## Step 3: Ability Contract (Actor Only)

Actor models must include:

- explicit `type Ability`,
- overridden `can()` with branch-level validation,
- `this.iCan(...)` wrapper usage.

Branch contract:

- validate ownership/facts,
- throw stable denial keys,
- avoid side effects inside permission checks.

## Step 4: ORM to GQL Mapping

If GraphQL touches the model:

- identify all bridge fields used in sort/filter/extras/unions,
- ensure fields are loaded by ORM (`requiredOrmAttrs` or includes),
- keep list queries bounded and ordered,
- keep depth and include controls active.

## Step 5: Validation

Run through:

- `docs/platforms/backend/patterns/orm-model-compliance-checklists.md`

Reject and revise if any mandatory item fails.

## Templates

### Actor Model Template (Skeleton)

```typescript
type Attrs = { id: number };
type Ability = { ACTION: { sub: "x" } };

export default class XModel extends Model<Attrs, Omit<Attrs, "id">, Ability> {
  static attributes() { return {}; }
  static initOptions() { return { modelName: "x", tableName: "xs" }; }
  static async boot() {}

  async can<To extends keyof Ability, CanProps extends CanPropsBase<Ability[To]>>(to: To, props: CanProps): Promise<Able> {
    return this.iCan(async () => {
      // branch checks
    }, props);
  }
}
```

### Non-Actor Model Template (Skeleton)

```typescript
type Attrs = { id: number };

export default class XModel extends Model<Attrs, Omit<Attrs, "id">> {
  static attributes() { return {}; }
  static initOptions() { return { modelName: "x", tableName: "xs" }; }
  static async boot() {}
}
```

## Refusal Conditions

Refuse to finalize and ask for clarification if:

- actor/non-actor classification is ambiguous,
- bridge relies on attrs not mapped in ORM loading strategy,
- ownership semantics are unclear for a permissioned operation,
- requested style conflicts with Ejtmaa baseline.

## Outputs

- Return changed file list and short rationale for each file.
- Include short change rationale with rejected alternatives when a structural choice matters.
- Explicitly state actor/non-actor classification decision.
- Explicitly state ORM-to-GQL mapping checks performed.

## Quality Gates

- style gate: Ejtmaa conventions only
- security gate: permission checks stay in requester/model boundaries
- mapping gate: no bridge field depends on an unloaded ORM attr
- relation gate: ownership and traversal direction remain consistent
- acceptance gate: mandatory checklist items pass before finalization

## Failure Handling

If partially blocked:

1. return the completed portion
2. list the exact missing inputs
3. provide the smallest follow-up question set
4. do not invent unresolved architecture decisions

## Ejtmaa Model Addendum (Strict)

Current non-actor tenant reference: `Organization` (`docs/platforms/backend/contracts/organization-domain.md`).

- Attrs comment groups: `//relations`, `//info`, `//virtual`; keep `attributes()` order aligned.
- Single-owner FK to Customer uses `customer_id` with a real constraint and unique index when cardinality is one-to-one.
- Do not use polymorphic `owner_*` for Organization unless product ownership changes.
- `Member` is non-actor under Organization (`organization_id` real FK). PK is UUID v4 (`id`); link/access uses separate unique `access_token` (UUID default) — no `public_id`. Customer is not a Member row.
- `MessageTemplate` is non-actor under Organization (`organization_id` real FK). Channel enum `messageTemplateChannel`: `WHATSAPP` | `EMAIL`. `subject` nullable (email); `body` required TEXT.
- `Meeting` is non-actor under Organization. UUID PK; `chairperson_id` → Member; optional `whatsapp_template_id` / `email_template_id` → MessageTemplate (`as` aliases). Enums: `meetingType`, `meetingStatus` (default `DRAFT`), `meetingNotifyStatus` (default `NOT_STARTED`); `notify_start_at` nullable. No media platform/url columns (LiveKit later).
- `MeetingParticipant` is roster join (composite PK `(meeting_id, member_id)` — no surrogate `id`). `Meeting.hasMany(..., { as: "participants" })` + `Member.hasMany(MeetingParticipant)`; join `belongsTo(Meeting)` + `belongsTo(Member)`. **No** `belongsToMany`. Type enum `meetingParticipantType`: `CHAIRPERSON` | `MEMBER` | `VIEWER`. Delivery: `notified` + `meetingParticipantDeliveryStatus` (default `PENDING`). Attendance (self check-in): optional `attended_at` + optional `left_at` (no `first_online_at` / `attended` boolean). Session permissions are fixed per type in product contract (`meeting-participant-domain.md` §8), not JSONB on the row.
- `AgendaItem` is non-actor under Meeting (`modelName: "agendaItem"` → default `hasMany` `agendaItems`, no `as`). UUID PK; `meeting_id`, `sort_order`, `subject`. Durable SQL agenda (not Yjs).
- `Decision` is non-actor under Meeting (`modelName: "decision"` → default `hasMany` `decisions`, no `as`). UUID PK; `meeting_id`, `sort_order`, `subject`, `phase` (`PRE_START`|`DURING`), `status` (default `NEW`), optional `voting_type` (`LIVE`|`SECRET`). Enums: `decisionPhase`, `decisionStatus`, `decisionVotingType`.
- `Vote` is durable ballot under Decision (composite PK `(decision_id, member_id)`; `modelName: "vote"` → `Decision.hasMany` `votes`, no `as`). Also `meeting_id` + `value` (`YES`|`NO` via `voteValue`) + `cast_at`. `belongsTo` Decision / Member / Meeting.
- `TalkRecord` is durable talk queue/history under Meeting (`modelName: "talkRecord"` → `Meeting.hasMany` `talkRecords`, no `as`). UUID PK; `meeting_id`, `member_id`, optional `sort_order`, `status` (`talkRecordStatus`: `WAITING`|`SPEAKING`|`COMPLETED`|`CANCELED`, default `WAITING`), `started_at`, optional `ended_at`. `belongsTo` Meeting / Member. No `decision_id`.
- `Plan` is non-actor platform catalog tier (الباقة; not org-owned). Do not name this model `Package` (language-sensitive). BIGINT PK; MultiLang `name` / optional `description`; dual SAR prices `monthly_price` + `yearly_price` (no single `price` / no `billing_period` on Plan); optional `max_members` / `max_meetings_per_month` (null = unlimited); `sort_order`; `status` (`planStatus`: `ACTIVE`|`DISABLED`, default `ACTIVE`). No `code` SKU field. No payment/subscription state on this model. Seed helper arrays must still not be named `plan`/`plans` (see demo-seed conventions). Attrs groups: `//info` only (no tenant FK).
- `Subscription` is non-actor, owned by **Customer** (`customer_id` real FK — not Organization). Also `plan_id` → Plan. Snapshot fields: `plan_price`, `plan_billing_period` (`planBillingPeriod`), `plan_billing_period_count`, optional `plan_max_members` / `plan_max_meetings_per_month`. Lifecycle columns: `starts_at`, `ends_at`, `status` (`subscriptionStatus`: `ACTIVE`|`EXPIRED`|`REPLACED`, default `ACTIVE`). No `PENDING`/`CANCELED`. Attrs groups: `//relations`, `//info`. `Customer.hasMany(Subscription)`; `Customer.hasOne(Subscription.scope("current"), { as: "currentSubscription" })` where scope `current` is a **function** (`ACTIVE` + `ends_at > now`); `Plan.hasMany(Subscription)`. Current catalog Plan is `currentSubscription.plan`. Hourly `ExpireSubscriptionsTask` promotes overdue `ACTIVE` rows to `EXPIRED`. **No** static subscribe/lifecycle helpers on the model — write flows belong in requesters when designed. No MyFatoorah fields on this model. Contract: `docs/platforms/backend/contracts/subscription-domain.md`; rule: `.cursor/rules/subscription-domain.mdc`.
