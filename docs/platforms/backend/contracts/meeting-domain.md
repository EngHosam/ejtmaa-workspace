# Meeting Domain Contract (Current)

## 1) Scope

Current Ejtmaa meeting surface:

- ORM persistence for organization-owned meetings,
- optional FKs `whatsapp_template_id` / `email_template_id` → `MessageTemplate` rows (legacy column names; template kinds are `messageTemplateType` — see `message-template-domain.md`; no inline template text on the meeting),
- chairperson FK to `Member`,
- independent lifecycle (`status`) and invite-notify axes (`notify_status` + `notify_start_at`),
- customer GraphQL read of meetings for the authenticated customer's organization,
- optional server-side list filter `_MeetingFilter` on root `meetings` (`search` → `subject` iLike; `status` → `_MeetingStatusValue` equality),
- website Meeting write path via `MeetingRequester` (`create` only) + `Customer.Ability.MEETING` — see §9,
- nested roster via `_Meeting.participants` (see `meeting-participant-domain.md`),
- nested agenda via `_Meeting.agendaItems` (see `agenda-item-domain.md`),
- nested decisions via `_Meeting.decisions` (see `decision-domain.md`),
- nested talk queue via `_Meeting.talkRecords` (see `talk-record-domain.md`),
- website GQL mirrors for that customer surface.

Out of scope (not shipped):

- meeting update / delete requesters,
- LiveKit join requesters / website client wiring (helper shipped — see `livekit-media-plane.md`),
- Yjs collaborative session state,
- supervisor Meeting GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for meetings,
- nested `_Organization.meetings` (B15 — root list only),
- `report_snapshot` / report materialization column,
- plan `max_meetings_per_month` quota enforcement on create.

## 2) Domain purpose

`Meeting` is a **non-actor** scheduled session record inside an `Organization`.

- Media is **LiveKit** at runtime — room/token helper shipped (`livekit-media-plane.md`); not modeled as Zoom/Teams `platform` / external `url` columns.
- Invite copy comes from optional template FKs, not duplicated body/subject fields on the meeting.
- Invite **start time** is `notify_start_at` (independent of `status`).
- Invite **progress** is `notify_status` (`NOT_STARTED` | `WAITING_TO_NOTIFY` | `NOTIFIED`).
- Lifecycle is `status` (`DRAFT` | `WAITING_TO_START` | `STARTED` | `COMPLETED` | `CANCELED`).

Tenant boundary: `organization_id`.

## 3) ORM model

File: `backend/src/app/orm/models/Meeting.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `meeting`
- `tableName`: `meetings`

### 3.1 Attrs layout

- `//relations` — `organization_id`, `chairperson_id`, `whatsapp_template_id`, `email_template_id`
- `//info` — subject, type, datetime, min_members_count, status, notify_status, notify_start_at

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | UUID PK | no | default `UUIDV4` |
| `organization_id` | BIGINT | no | FK → Organization |
| `chairperson_id` | UUID | no | FK → Member.id (`as: "chairperson"`) |
| `whatsapp_template_id` | BIGINT | yes | FK → MessageTemplate (`as: "whatsappTemplate"`) |
| `email_template_id` | BIGINT | yes | FK → MessageTemplate (`as: "emailTemplate"`) |
| `subject` | STRING(191) | no | |
| `type` | STRING(191) | no | enum `meetingType` |
| `datetime` | DATE | no | scheduled time |
| `min_members_count` | INTEGER | no | quorum hint |
| `status` | STRING(191) | no | enum `meetingStatus`; default `DRAFT` |
| `notify_status` | STRING(191) | no | enum `meetingNotifyStatus`; default `NOT_STARTED` |
| `notify_start_at` | DATE | yes | when invite sending may begin |

Exported TS types: `MeetingType`, `MeetingStatus`, `MeetingNotifyStatus` from `G_Tr` enum keys.

### 3.3 Enums (localized)

Under `backend/src/resources/trans/ar/general.ts` and `en/general.ts`:

| Enum key | Values |
|---|---|
| `meetingType` | `PERIODIC`, `EMERGENCY` |
| `meetingStatus` | `DRAFT`, `WAITING_TO_START`, `STARTED`, `COMPLETED`, `CANCELED` |
| `meetingNotifyStatus` | `NOT_STARTED`, `WAITING_TO_NOTIFY`, `NOTIFIED` |

### 3.4 Indexes

- `meetings_organization_id` — list by org

### 3.5 Relations

`Meeting.boot()`:

- `belongsTo(Organization)` on `organization_id`
- `belongsTo(Member, { as: "chairperson" })` on `chairperson_id`
- `belongsTo(MessageTemplate, { as: "whatsappTemplate" })` on `whatsapp_template_id`
- `belongsTo(MessageTemplate, { as: "emailTemplate" })` on `email_template_id`
- `hasMany(MeetingParticipant, { as: "participants" })` on `meeting_id` (roster; see `meeting-participant-domain.md`)
- `hasMany(AgendaItem)` on `meeting_id` (default association `agendaItems`, no `as`; see `agenda-item-domain.md`)
- `hasMany(Decision)` on `meeting_id` (default association `decisions`, no `as`; see `decision-domain.md`)
- `hasMany(TalkRecord)` on `meeting_id` (default association `talkRecords`, no `as`; see `talk-record-domain.md`)

`Organization.boot()` inverse:

- `hasMany(Meeting)` on `organization_id`
- mixins: `getMeetings` / `createMeeting` / … (association PK type `string` for UUID meeting id)

Mixin declare blocks on Meeting are split: organization / chairperson / whatsapp template / email template / participants / agenda items / decisions / talk records.

Do **not** add `belongsToMany(Member)` on Meeting for the roster — join rows are exposed via `participants` only.

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MeetingType`, `_MeetingStatus`, `_MeetingNotifyStatus` (+ Value enums)
- `backend/src/app/gql/definitions/customer.graphql` — `_Meeting` + `_MeetingFilter` + roots

### Type `_Meeting`

Implements `_Timestamps` & `_Pagination`.

Info: `id`, `subject`, `type`, `datetime`, `min_members_count`, `status`, `notify_status`, `notify_start_at`.

Relations:

- `organization: _Organization` (cardinality-safe `belongsTo`)
- `chairperson: _Member` (association `as` = field name)
- `whatsappTemplate: _MessageTemplate`
- `emailTemplate: _MessageTemplate`
- `participants: [_MeetingParticipant]` (roster nest; B15 OK for expected board size — contract in `meeting-participant-domain.md`)
- `agendaItems: [_AgendaItem]` (agenda nest; contract in `agenda-item-domain.md`)
- `decisions: [_Decision]` (decision nest; contract in `decision-domain.md`)
- `talkRecords: [_TalkRecord]` (talk-queue nest; contract in `talk-record-domain.md`)

### Root queries

- `meetings(filter: _MeetingFilter): [_Meeting]`
- `meeting(id: ID!): _Meeting`

Filter input:

```graphql
input _MeetingFilter {
    search: String
    status: _MeetingStatusValue
}
```

Resolvers (`CustomerSchema`):

```ts
prepareManyGQLModels({ me: true, filter: filter || undefined })
prepareOneGQLModel({ me: true, id })
```

### Bridge: `MeetingBridge`

File: `backend/src/app/gql/bridges/customer/MeetingBridge.ts`

- Extends `CustomerOrganizationOwnedBridgeBase`
- `ident = "meeting"`, `typeIdent = "_Meeting"`, `ormModel = MeetingModel`
- `MeetingFilter` / `GetManyParent = OrganizationOwnedMeParent & { filter?: Nullable<MeetingFilter> }`
- `GetOneParent = MeetingModel | { me: true; id: string }`
- Overrides `getOrmFindOptions` for root `many` only:
  - When `filter.search` trims non-empty, adds `subject` `iLike`
  - When `filter.status` is set, adds `status` equality (`_MeetingStatusValue`)
  - Always `withListable` + `withReplacements` + `order updated_at DESC`
  - Otherwise delegates to `super`

Parent-payload discipline: `.cursor/rules/gql-root-parent-payload-contract.mdc` (§3 — filter mapping is entity-owned policy).

### Inverse / nested parent typing (mandatory)

| Nested SDL field | Preparing bridge | Parent typing |
|---|---|---|
| `_Meeting.organization` | `OrganizationBridge` | `GetOneParent` includes `MeetingModel` |
| `_Meeting.chairperson` | `MemberBridge` | `GetOneParent` includes `MeetingModel` |
| `_Meeting.whatsappTemplate` / `emailTemplate` | `MessageTemplateBridge` | `GetOneParent` includes `MeetingModel` |
| `_Meeting.participants` | `MeetingParticipantBridge` | `GetManyParent = MeetingModel` |
| `_Meeting.agendaItems` | `AgendaItemBridge` | `GetManyParent = MeetingModel` |
| `_Meeting.decisions` | `DecisionBridge` | `GetManyParent = MeetingModel` |
| `_Meeting.talkRecords` | `TalkRecordBridge` | `GetManyParent = MeetingModel` |

Current shapes:

```ts
// OrganizationBridge
export type GetOneParent =
    | CustomerModel
    | MemberModel
    | MessageTemplateModel
    | MeetingModel;

// MemberBridge
export type GetOneParent =
    | MemberModel
    | MeetingModel
    | MeetingParticipantModel
    | VoteModel
    | TalkRecordModel
    | { me: true; id: string };

// MessageTemplateBridge
export type GetOneParent = MessageTemplateModel | MeetingModel | { me: true; id: string };
```

### Registered bridges

`CustomerSchema.registeredBridges` includes `MeetingBridge`, `MeetingParticipantBridge`, `AgendaItemBridge`, `DecisionBridge`, `VoteBridge`, and `TalkRecordBridge`.

## 5) Read flow (root)

### `meetings`

1. `MeetingBridge.AsRoot` → `prepareManyGQLModels({ me: true, filter })`
2. `CustomerOrganizationOwnedBridgeBase.getRootOrmParent` → customer's Organization
3. List: `withListable` + `withReplacements` + `order updated_at DESC` + optional `_MeetingFilter` where (`search` → `subject` iLike; `status` → equality)
4. Organization → meetings association — tenant scope remains the association; filter only narrows within the org

### `meeting(id)`

1. `prepareOneGQLModel({ me: true, id })`
2. Same org resolve
3. `_MeetingFilter` does **not** apply to the singular root
3. Base one: `where: { id }` scoped to that organization

## 6) Seed

No meeting seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — base + customer SDL/types synced |
| `cpanel/` | Deferred — no supervisor Meeting surface |

Verification: `yarn generate-types`, `yarn type-check`.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `meetings` / `meeting` | no `context.customer` | `NOT_PERMIT` |
| `meetings` / `meeting` | customer has no organization | `404` |
| `meeting(id)` | missing / other-org id | framework empty → `404` |

## 9) Customer write path (create)

### 9.1 Ability

File: `backend/src/app/orm/models/Customer.ts`

```ts
MEETING: {
    sub: "create"
}
```

`can("MEETING", { sub: "create" })`: requires organization; otherwise `ACTION_NOT_ALLOWED`. No plan quota gate in this slice.

GQL UI exposure (same rule, `visualMode`):

- SDL: `_Me.canCreateMeeting: _Ability`
- Bridge: `MeBridge` extra `canCreateMeeting` → `me.can("MEETING", { sub: "create", visualMode: { lang } })`

### 9.2 Joi

- `isCustomerOwnedMember(joi, path?, optional?)` — `path` defaults to `"member"` (same order/name as `Model().Opt`); meeting create uses `"chairperson"`.
- Create fields: `subject` trim min 2; `type` `PERIODIC` | `EMERGENCY`; `datetime` date; `min_members_count` positive int (accepts numeric string); `chairperson` owned member.

### 9.3 Requester

File: `backend/src/app/orchestrator/requesters/MeetingRequester.ts` (`@requester("meeting")`, platform `website`, actor `customer`).

| Sub | Behavior |
|---|---|
| `create` | Validate → `can MEETING create` → `organization.createMeeting(...)` (defaults `status=DRAFT`, `notify_status=NOT_STARTED`) → `meeting.createParticipant({ type: "CHAIRPERSON", notified: false, delivery_status: "PENDING" })` → `other.meetingId` + `SUCCESS_CREATE` |

Create payload (after Joi / Opt resolution):

| Field | Joi | Persist |
|---|---|---|
| `subject` | `string().trim().min(2)` | `meeting.subject` |
| `type` | `PERIODIC` \| `EMERGENCY` (select) | `meeting.type` |
| `datetime` | `joi.date()` (any date; **no** server future check) | `meeting.datetime` |
| `min_members_count` | `number().integer().min(1)` | `meeting.min_members_count` |
| `chairperson` | `isCustomerOwnedMember(joi, "chairperson")` | `chairperson_id` + roster `CHAIRPERSON` row |

Website UI consumer: `docs/platforms/website/flow-customer-meetings.md` §5.

### 9.4 Maps

| Map | Entry |
|---|---|
| `backend/requesters.website.ts` | `customer.meeting: "create"` |
| `website/src/types/requesters/requesters.website.ts` | Same union |

### 9.5 Failure modes (write)

| Condition | Result |
|---|---|
| No org on create (`can`) | `ACTION_NOT_ALLOWED` |
| Org missing after `can` succeeded | `UNEXPECTED_ERROR` |
| Other-org / missing chairperson | `NOT_PERMIT` / Opt `404` |
| Validation errors | field / firewall errors via `throwIfNotValid` |

Verify: `yarn type-check` in `backend/`.

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/gql/bridges/customer/MeBridge.ts` | `canCreateMeeting` visualMode extra | §9.1 |
| `backend/src/app/orm/models/Customer.ts` | `Ability.MEETING` + `can("MEETING")` | §9.1 |
| `backend/src/app/validation/joi_rules.ts` | `isCustomerOwnedMember` (optional `prop`) | §9.2 |
| `backend/src/app/orchestrator/requesters/MeetingRequester.ts` | write requester (`create`) | §9.3 |
| `backend/requesters.website.ts` | `customer.meeting` map | §9.4 |
| `website/src/types/requesters/requesters.website.ts` | `customer.meeting` mirror | §9.4 |
| `backend/src/app/orm/models/Meeting.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/MeetingParticipant.ts` | Roster join (detail contract) | `meeting-participant-domain.md` |
| `backend/src/app/orm/models/AgendaItem.ts` | Agenda lines (detail contract) | `agenda-item-domain.md` |
| `backend/src/app/orm/models/Decision.ts` | Decisions (detail contract) | `decision-domain.md` |
| `backend/src/app/orm/models/TalkRecord.ts` | Talk queue (detail contract) | `talk-record-domain.md` |
| `backend/src/app/orm/models/Organization.ts` | `hasMany Meeting` + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | meeting enums AR | §3.3 |
| `backend/src/resources/trans/en/general.ts` | meeting enums EN | §3.3 |
| `backend/src/app/gql/definitions/base.graphql` | meeting GQL enum wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_Meeting` + `_MeetingFilter` + roots + nested relations | §4 |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | thin org-owned bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/MeetingParticipantBridge.ts` | nested roster bridge | `meeting-participant-domain.md` |
| `backend/src/app/gql/bridges/customer/AgendaItemBridge.ts` | nested agenda bridge | `agenda-item-domain.md` |
| `backend/src/app/gql/bridges/customer/DecisionBridge.ts` | nested decision bridge | `decision-domain.md` |
| `backend/src/app/gql/bridges/customer/TalkRecordBridge.ts` | nested talk-record bridge | `talk-record-domain.md` |
| `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | shared `me` → Organization | §4 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | inverse parent typing | §4 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | chairperson + participant/vote/talkRecord.member parent typing | §4 |
| `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts` | template parent typing | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | register + resolvers | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enums); no meeting roots | §7 |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry (gitignored) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `docs/platforms/backend/contracts/agenda-item-domain.md`
- `docs/platforms/backend/contracts/decision-domain.md`
- `docs/platforms/backend/contracts/talk-record-domain.md`
- `docs/platforms/backend/contracts/livekit-media-plane.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/website/flow-customer-meetings.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `docs/invariants/backend.md` (B15)
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
- `.cursor/rules/organization-tenant-ownership.mdc`
