# Meeting Domain Contract (Current)

## 1) Scope

Current Ejtmaa meeting surface:

- ORM persistence for organization-owned meetings,
- optional FKs `whatsapp_template_id` / `email_template_id` → `MessageTemplate` rows (legacy column names; template kinds are `messageTemplateType` — see `message-template-domain.md`; no inline template text on the meeting),
- chairperson FK to `Member`,
- independent lifecycle (`status`) and invite-notify axes (`notify_status` + `notify_start_at`),
- schedule policy on the write path: 10-minute minimum lead on `datetime`, a client-chosen `notify_start_at` that must be in the future and at least 5 minutes before `datetime`, and a hard edit lock once the meeting leaves `DRAFT` (§9.1a),
- customer GraphQL read of meetings for the authenticated customer's organization,
- optional server-side list filter `_MeetingFilter` on root `meetings` (`search` → `subject` iLike; `status` → `_MeetingStatusValue` equality),
- website Meeting write path via `MeetingRequester` (`create` | `read` | `update` | `updateTemplates` | `delete` | `approve` | `cancel` + child roster/agenda/decision subs) + `Customer.Ability.MEETING` — see §9,
- bounded `_Meeting` Ability extras `canUpdate` / `canDelete` / `canApprove` / `canCancel` (detail only),
- nested roster via `_Meeting.participants` (see `meeting-participant-domain.md`),
- nested agenda via `_Meeting.agendaItems` (see `agenda-item-domain.md`),
- nested decisions via `_Meeting.decisions` (see `decision-domain.md`),
- nested talk queue via `_Meeting.talkRecords` (see `talk-record-domain.md`),
- live collaborative session state for `subject` / `type` / `status` / `participants` in the `live_state` BLOB (see `meeting-live-state.md`),
- website GQL mirrors for that customer surface.

Out of scope (not shipped):

- LiveKit client `Room.connect` / A/V UI (join-token HTTP + participant token cache shipped — see `livekit-media-plane.md` §6),
- reflecting the live session fields back onto the SQL columns (`meeting-live-state.md` §6),
- starting notify inside `approve` (scheduler owns claim — `meeting-invite-notify.md`),
- supervisor Meeting GraphQL,
- cpanel mirrors exist under `cpanel/src/types/gql/**`; bootstrap UI does not consume this domain,
- seed rows for meetings,
- nested `_Organization.meetings` (B15 — root list only),
- `report_snapshot` / report materialization column,
- plan `max_meetings_per_month` quota enforcement on create.

## 2) Domain purpose

`Meeting` is a **non-actor** scheduled session record inside an `Organization`.

- Media is **LiveKit** at runtime — room/token helper shipped (`livekit-media-plane.md`); not modeled as Zoom/Teams `platform` / external `url` columns.
- Invite copy comes from optional template FKs, not duplicated body/subject fields on the meeting.
- Invite **start time** is `notify_start_at` (independent of `status`). The customer picks it on create and on basics update; Joi requires it to be in the future and at least `MeetingModel.NOTIFY_MIN_GAP_MS` before `datetime`. Approve does not re-derive it.
- Invite **progress** is `notify_status` (`NOT_STARTED` | `WAITING_TO_NOTIFY` | `NOTIFIED` | `PARTIALLY_NOTIFIED` | `FAILED`). Send pipeline: `meeting-invite-notify.md`.
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
- `//live session` — `live_state`

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
| `notify_start_at` | DATE | yes | when invite sending may begin; chosen by the customer on create/update (nullable for rows written before the field shipped) |
| `live_state` | BLOB | yes | Yjs V2 snapshot of the live session document; never exposed through GraphQL (`meeting-live-state.md`) |

Exported TS types: `MeetingType`, `MeetingStatus`, `MeetingNotifyStatus` from `G_Tr` enum keys. Live CRDT map shape: `MeetingLiveMap` in `backend/src/app/types/meeting.ts` (mirrored on website).

### 3.2b Schedule constants

Statics on `MeetingModel` are the single source for the numbers they own. The two schedule windows are independent: `MIN_LEAD_MS` bounds how soon `datetime` may be, and `NOTIFY_MIN_GAP_MS` bounds how close `notify_start_at` may sit to `datetime`. Keep `NOTIFY_MIN_GAP_MS` below `MIN_LEAD_MS`, otherwise a meeting created at the minimum lead has no valid invite start; the tail of `Meeting.ts` runs a self-check that throws on boot if that ordering is ever broken. Ability, Joi, and the requester read the statics; nothing re-declares those. The attend-open constant is authority for the live self-check-in window (website client gate today).

| Static | Value | Used by |
|---|---|---|
| `NOTIFY_MIN_GAP_MS` | `5 * 60 * 1000` | `isMeetingNotifyStartAt` Joi rule — `notify_start_at ≤ datetime − NOTIFY_MIN_GAP_MS` (§9.1a) |
| `MIN_LEAD_MS` | `10 * 60 * 1000` | `isMeetingDatetime` Joi rule (§9.1a) |
| `ATTEND_OPEN_BEFORE_MS` | `30 * 60 * 1000` | Self-check-in may open at `datetime − ATTEND_OPEN_BEFORE_MS`; website mirror private inside `useMeetingAttendWindow.ts`; session exposes `attendWindow` and feeds `windowOpen` into `can.attend` (`organization-host-routing.md` §5.3; `meeting-participant-domain.md` §3.6). Not a Joi/Ability write gate. |

Website schedule-lead UI mirrors (when present) stay copy/client validation only (`flow-customer-meetings.md` §6.5). The attend-window mirror lives in `website/src/app/ui/components/meeting/hooks/useMeetingAttendWindow.ts` (session-owned clock; screens read `attendWindow`).

### 3.3 Enums (localized)

Under `backend/src/resources/trans/ar/general.ts` and `en/general.ts`:

| Enum key | Values |
|---|---|
| `meetingType` | `PERIODIC`, `EMERGENCY` |
| `meetingStatus` | `DRAFT`, `WAITING_TO_START`, `STARTED`, `COMPLETED`, `CANCELED` |
| `meetingNotifyStatus` | `NOT_STARTED`, `WAITING_TO_NOTIFY`, `NOTIFIED`, `PARTIALLY_NOTIFIED`, `FAILED` |

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

### 3.6 Live session members

The ORM keeps the `live_state` column; codec/seed/registry live in `MeetingLiveDocHelper`; `MEETING_LIVE_MAP` / `MEETING_LIVE_STATUSES` live in the mirrored `types/meeting.ts`. Full contract: `meeting-live-state.md` §1.

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
- `registerOrmAttrs = { expect: ["live_state"] }` — excludes the CRDT BLOB from the auto-registered attrs (`meeting-live-state.md` §4)
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

### 7.1 Realtime surface

Meeting realtime is not part of the customer GQL surface. It is socket namespace `/meeting`, entered by a `Member` with `access_token` plus a `MeetingParticipant` roster row, and it joins room `Rooms.MEETING(meetingId)`. Events `meeting.live.sync` / `meeting.live.update` / `meeting.live.error` carry a Yjs document for `subject`, `type`, `status`, and `participants`, persisted in `live_state`.

While a meeting is live those map fields are authoritative **in the document**, not in the columns; the columns still hold what the requester write path left, and the reflection step is deferred. Contracts: `docs/platforms/backend/contracts/meeting-realtime-socket.md` (transport) and `docs/platforms/backend/contracts/meeting-live-state.md` (state); website consumer: `docs/platforms/website/organization-host-routing.md` §5.1.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `meetings` / `meeting` | no `context.customer` | `NOT_PERMIT` |
| `meetings` / `meeting` | customer has no organization | `404` |
| `meeting(id)` | missing / other-org id | framework empty → `404` |

## 9) Customer write path (create / update / approve / children)

### 9.1 Ability

File: `backend/src/app/orm/models/Customer.ts`

```ts
MEETING: {
    sub: "create" | "read" | "update" | "delete" | "approve"
    meeting?: ModelRef<MeetingModel>
}
```

| sub | Gate |
|---|---|
| `create` | org required |
| `read` | org-owned meeting |
| `update` | org-owned + `DRAFT` + `notify_status === NOT_STARTED` — covers basics, templates, roster, agenda, decisions (no separate `updateTemplates` Ability sub) |
| `delete` | org-owned + `DRAFT` + `NOT_STARTED` |
| `approve` | org-owned + **active subscription** (`Customer.getCurrentSubscription`) + `DRAFT` + `NOT_STARTED` + invite start still ahead (§9.1a) + completeness (§9.1b) |
| `cancel` | org-owned + status is `DRAFT` or `WAITING_TO_START` (never after the session started) |

GQL UI exposure (`visualMode`):

- `_Me.canCreateMeeting` — MeBridge (unchanged)
- `_Meeting.canUpdate` / `canDelete` / `canApprove` / `canCancel` — `MeetingBridge.loadExtra` (bounded root-one only)

Helper: `backend/src/app/helpers/meetingNotifyTemplateMode.ts` (`resolveMeetingNotifyTemplateMode`, satisfy + denial keys). Website mirrors the mode resolver under `meetings/meetingNotifyTemplateMode.ts` for callout copy only — enforcement stays in Ability.

When Ability loads the roster for the notify-template mode gate, it uses `meeting.getParticipants({ include: ["member"] })` — association **name** string, not `Member()` model class (same pattern as live-doc seed in `meeting-live-state.md` §1.3; `.cursor/rules/sequelize-include-by-association-name.mdc`).

### 9.1a Schedule policy (lead, invite start, approval finality)

Schedule rules run on the write path. All of them are Ability- or Joi-owned; the website only mirrors them for copy.

| Rule | Where | Condition | Denial |
|---|---|---|---|
| Minimum lead on write | `isMeetingDatetime(joi)` on `create` / `update` | `datetime >= now + MIN_LEAD_MS` | Joi `datetime.tooSoon` |
| Invite start in the future | `isMeetingNotifyStartAt(joi)` on `create` / `update` | `notify_start_at >= now` | Joi `notify_start_at.past` |
| Invite start gap | `isMeetingNotifyStartAt(joi)` on `create` / `update` | `notify_start_at <= datetime − NOTIFY_MIN_GAP_MS` | Joi `notify_start_at.tooLate` |
| Invite start not missed | `Ability.MEETING` `approve` | `notify_start_at >= now` | `MEETING_NOTIFY_TIME_PASSED` |
| Editing window | `Ability.MEETING` `update` | `status === DRAFT` | `MEETING_NOT_DRAFT` |

`notify_start_at` is **client-owned**: the create form and the basics modal both render it, and the requester persists the validated value verbatim. Nothing derives it from `datetime` on the write path. `read` falls back to `datetime − NOTIFY_MIN_GAP_MS` only to hydrate rows written before the column became client-owned.

Approval is final. Once `approve` moves a meeting to `WAITING_TO_START`, every content write — basics, templates, roster, agenda, decisions — is denied with `MEETING_NOT_DRAFT`; there is no demotion back to `DRAFT`. The escape hatch is `cancel` (§9.3), available until the session starts. That removes the approve/demote cycle that previously churned `live_state`.

`isMeetingNotifyStartAt` reads its sibling through `smartHelpers.get("datetime")`, so it only works in a schema that also carries `datetime`.

### 9.1b Approve completeness

0. **Active subscription** — `Customer.getCurrentSubscription` (ACTIVE + not ended scope). Missing → `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED`. Create/update draft remains allowed without a subscription.
1. `notify_start_at ≥ now` (§9.1a) — the invite window has not been missed.
2. **Voting** roster count ≥ `min_members_count` — `countParticipants` filters `type IN (CHAIRPERSON, MEMBER)`; `VIEWER` rows never count toward quorum.
3. ≥1 agenda item.
4. ≥1 decision with `phase === PRE_START`.
5. Notify templates per contact mode on roster members’ `email` / `mobile`:

| Mode | Rule |
|---|---|
| `MISSING_CONTACT` | any participant lacks both → deny |
| `EMAIL_ONLY` | every has email → `email_template_id` required |
| `WHATSAPP_ONLY` | every has mobile → `whatsapp_template_id` required |
| `BOTH` | mixed coverage → both FKs required |
| `ANY_ONE` | every has both → at least one FK |

WhatsApp FK types: `ADWHATS` \| `ADWHATS_PRO`. Email FK types: `EJTMAA_EMAIL` \| `CUSTOM_EMAIL`.

Approve writes `status = WAITING_TO_START` and clears `live_state`, registering `afterCommit` → `destroyMeetingLiveDoc(meetingId, { flush: false })`. It keeps the client-chosen `notify_start_at` and does **not** change `notify_status`. Claim and send are `InviteNotifyTask` (`meeting-invite-notify.md`).

### 9.1c Active subscription outside approve (live entry)

The same helper and denial key also gate **live entry**, not as approve-completeness steps:

| Surface | Where | Effect |
|---|---|---|
| Meeting socket handshake | `MeetingAuthenticationIOMiddleware` | Refuse connect (`MEETING_ACTIVE_SUBSCRIPTION_REQUIRED`) |
| LiveKit join token | `OrganizationHostMiddleware` (`org_host`) | Refuse before mint/reuse; `/custom/org/start` does not use `org_host` |

Website checkout / schedule-note UX: `docs/platforms/website/flow-customer-subscription.md` §11. Socket: `meeting-realtime-socket.md` §2 / §6.1. Token: `livekit-media-plane.md` §7.2.

### 9.2 Joi

- `isCustomerOwnedMember`, `isCustomerOwnedMeeting`, `isCustomerOwnedAgendaItem`, `isCustomerOwnedDecision`, `isCustomerOwnedMessageTemplate` in `joi_rules.ts`.
- `isMeetingDatetime(joi)` — `joi.date()` plus an `external` that rejects a non-date, an invalid date, or `datetime < now + MeetingModel.MIN_LEAD_MS` with the `datetime.tooSoon` Joi key (localized in `trans/{ar,en}/general.ts` under `joi`). Used by `create` and `update`; presence stays the field's own `required` semantics.
- `isMeetingNotifyStartAt(joi)` — `joi.date()` plus an `external` that rejects an invalid or past value (`notify_start_at.past`) and a value later than `datetime − MeetingModel.NOTIFY_MIN_GAP_MS` (`notify_start_at.tooLate`). It reads `datetime` from the same payload.
- Create/update basics: subject, type, datetime, notify_start_at, min_members_count, chairperson. Template FKs live in their own `updateTemplates` schema (meeting + two template refs) so a template change never re-validates the schedule. Everything stays mutable while the meeting is a `DRAFT`.
- The two template refs are optional through the helper flag only — `isCustomerOwnedMessageTemplate(joi, path, true)` with no chained `.allow(null)`, since `Model.Opt` already returns `optional().allow(null)` for that flag. A cleared picker arrives as `null`, `Model.Opt` passes it through as absent, and the requester writes the FK as `null`.

### 9.3 Requester

File: `backend/src/app/orchestrator/requesters/MeetingRequester.ts` (`@requester("meeting")`).

| Sub | Behavior |
|---|---|
| `create` | org create + CHAIRPERSON roster → `other.meetingId`; persists the client `notify_start_at` |
| `read` | hydrate SelectOption fields + `notify_start_at`; **no** `meeting` id echo |
| `update` | basics + `notify_start_at` (**no** template FKs); chair swap demotes previous chair to `MEMBER`, promotes/creates CHAIRPERSON |
| `updateTemplates` | template FKs only (`whatsappTemplate` \| `emailTemplate`, both nullable, kind-checked); gated by the `update` Ability |
| `delete` | destroy children then meeting (`force`) |
| `approve` | `DRAFT` → `WAITING_TO_START` + `live_state = null` + live-doc destroy (§9.1b) |
| `cancel` | `DRAFT` \| `WAITING_TO_START` → `CANCELED` + leftover `PENDING` → `CANCELED` + `notify_status = finalizeNotifyStatus` + `live_state = null` + live-doc destroy |
| `addParticipant` / `removeParticipant` | MEMBER\|VIEWER; cannot remove chairperson |
| `createAgendaItem` / `updateAgendaItem` / `deleteAgendaItem` | subject; `sort_order = max+1` on create |
| `createDecision` / `updateDecision` / `deleteDecision` | create: client `phase` PRE_START\|DURING, status NEW, `voting_type null`; update/delete phase-agnostic under prepare Ability (`notify_status === NOT_STARTED`) |

Every `update`-family sub in the table (basics, both participant subs, all agenda subs, all decision subs) gates on `can("MEETING", { sub: "update" })`, so approval alone closes all of them. No sub demotes an approved meeting back to `DRAFT`.

Website UI: `docs/platforms/website/flow-customer-meetings.md` §5–§6.

### 9.4 Maps

| Map | Entry |
|---|---|
| `backend/requesters.website.ts` | `customer.meeting` union of all subs above |
| website mirror | Same (W18) |

### 9.5 Failure modes

Write-path Ability/Joi denials, plus live-entry subscription refuses (§9.1c):

| Condition | Result |
|---|---|
| No org | `ACTION_NOT_ALLOWED` |
| Notify started on mutate | `MEETING_NOTIFY_STARTED` |
| `datetime` under 10-minute lead on create/update | Joi `datetime.tooSoon` (field error) |
| `notify_start_at` in the past | Joi `notify_start_at.past` (field error) |
| `notify_start_at` within 5 minutes of `datetime` | Joi `notify_start_at.tooLate` (field error) |
| No active subscription on approve | `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` |
| No active subscription on Meeting handshake / LiveKit `org_host` | `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` (§9.1c) |
| Invite start already passed on approve | `MEETING_NOTIFY_TIME_PASSED` |
| Update/approve/delete not draft | `MEETING_NOT_DRAFT` |
| Cancel after the session started | `MEETING_ALREADY_STARTED` |
| Quorum (voting types only) / agenda / decision / template gaps | dedicated `MEETING_*` message keys |
| Duplicate participant | `DUPLICATED` |
| Other-org / missing | `NOT_PERMIT` / `404` |

Verify: `yarn generate-types`, `yarn type-check` in `backend/`.

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/gql/bridges/customer/MeBridge.ts` | `canCreateMeeting` visualMode extra | §9.1 |
| `backend/src/app/orm/models/Customer.ts` | `Ability.MEETING`; org scope, draft-only editing, cancel window, approve invite-window + completeness + active subscription; roster include `["member"]` for notify-template mode | §9.1–§9.1b |
| `backend/src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | Meeting handshake active-subscription gate | §9.1c |
| `backend/src/app/http/middlewares/OrganizationHostMiddleware.ts` | `org_host` active-subscription gate (LiveKit token) | §9.1c |
| `backend/src/app/helpers/meetingNotifyTemplateMode.ts` | Contact-mode resolver, template satisfiability, denial keys, executable matrix self-check | §9.1b |
| `backend/src/app/validation/joi_rules.ts` | Customer-owned Meeting, agenda, decision, member, and template hydration; `isMeetingDatetime` lead rule; `isMeetingNotifyStartAt` invite-window rule | §9.2 |
| `backend/src/app/orchestrator/requesters/MeetingRequester.ts` | Basics, templates, approve, cancel, delete, participant, agenda, and decision requester subs; client-owned `notify_start_at`; cancel leftover + `finalizeNotifyStatus`; live-doc destroy on approve/cancel | §9.1a, §9.3; `meeting-invite-notify.md` |
| `backend/requesters.website.ts` | Backend customer `meeting` sub map | §9.4 |
| `website/src/types/requesters/requesters.website.ts` | Exact website customer `meeting` sub-map mirror | §9.4 |
| `backend/src/app/orm/models/Meeting.ts` | ORM source of truth; `NOTIFY_MIN_GAP_MS` / `MIN_LEAD_MS` / `ATTEND_OPEN_BEFORE_MS` statics + window self-check; `live_state` column | §3, §3.2b, §3.6 |
| `backend/src/app/scheduler/tasks/InviteNotifyTask.ts` | Invite claim + paced send | `meeting-invite-notify.md` |
| `backend/src/app/helpers/MeetingLiveDocHelper.ts` | Live document registry and BLOB persistence; `destroyMeetingLiveDoc` consumed by approve/cancel | `meeting-live-state.md` §2–§3 |
| `backend/src/app/socket/controllers/meeting/*` | `/meeting` connection and `meeting.live.*` controllers | `meeting-realtime-socket.md` §3 |
| `backend/src/app/orm/models/Member.ts` | `forSelect(lang)` used to hydrate chairperson and roster entity references | §9.2–§9.3 |
| `backend/src/app/orm/models/MessageTemplate.ts` | `forSelect(lang)` used to hydrate and validate notify template references | §9.2–§9.3 |
| `backend/src/app/orm/models/MeetingParticipant.ts` | Roster join (detail contract) | `meeting-participant-domain.md` |
| `backend/src/app/orm/models/AgendaItem.ts` | Agenda lines (detail contract) | `agenda-item-domain.md` |
| `backend/src/app/orm/models/Decision.ts` | Decisions (detail contract) | `decision-domain.md` |
| `backend/src/app/orm/models/TalkRecord.ts` | Talk queue (detail contract) | `talk-record-domain.md` |
| `backend/src/app/orm/models/Organization.ts` | `hasMany Meeting` + mixins | §3.5 |
| `backend/src/resources/trans/ar/general.ts` / `en/general.ts` | Meeting, decision, notify, and message-template enum labels; `joi.datetime.tooSoon`, `joi.notify_start_at.past` / `.tooLate` | §3.3, §9.1a, §9.1b |
| `backend/src/resources/trans/ar/messages.ts` / `en/messages.ts` | Ability denial messages for meeting completeness, active subscription (`MEETING_ACTIVE_SUBSCRIPTION_REQUIRED`), notify lock, draft-only edit (`MEETING_NOT_DRAFT`), cancel window (`MEETING_ALREADY_STARTED`), missed invite window (`MEETING_NOTIFY_TIME_PASSED`), voting-quorum copy | §9.1a, §9.1b |
| `backend/src/resources/trans/ar/validation.ts` / `eng-hosam/@nodejs/validation/src/trans/ar/validation.ts` | Arabic email validation labels | localization-only |
| `backend/src/app/gql/definitions/base.graphql` | meeting GQL enum wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_Meeting` + `_MeetingFilter` + roots + nested relations | §4 |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | Org-scoped root filters, root-one `canUpdate` / `canDelete` / `canApprove` / `canCancel` extras, `live_state` attr exclusion | §4–§5, §9.1 |
| `backend/src/app/gql/bridges/customer/MeetingParticipantBridge.ts` | nested roster bridge | `meeting-participant-domain.md` |
| `backend/src/app/gql/bridges/customer/AgendaItemBridge.ts` | nested agenda bridge | `agenda-item-domain.md` |
| `backend/src/app/gql/bridges/customer/DecisionBridge.ts` | nested decision bridge | `decision-domain.md` |
| `backend/src/app/gql/bridges/customer/VoteBridge.ts` / `TalkRecordBridge.ts` | Nested child bridge parent typing | `vote-domain.md` / `talk-record-domain.md` |
| `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | shared `me` → Organization | §4 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | inverse parent typing | §4 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | chairperson + participant/vote/talkRecord.member parent typing | §4 |
| `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts` | Template parent typing and `type` / `types[]` picker filters | §4, §9.2 |
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
- `docs/platforms/backend/contracts/meeting-realtime-socket.md`
- `docs/platforms/backend/contracts/meeting-live-state.md`
- `docs/platforms/backend/contracts/meeting-invite-notify.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/website/flow-customer-meetings.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `docs/invariants/backend.md` (B15, B27)
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
- `.cursor/rules/organization-tenant-ownership.mdc`
