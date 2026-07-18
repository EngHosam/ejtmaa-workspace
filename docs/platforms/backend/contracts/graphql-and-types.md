# GraphQL and Types Contract (Ejtmaa)

## Schemas

Provider: `backend/src/resources/configs/gql/index.ts`

| Schema | SDL | Schema class | Role |
|---|---|---|---|
| `customer` | `backend/src/app/gql/definitions/customer.graphql` | `backend/src/app/gql/schemas/CustomerSchema.ts` | Customer actor reads |
| `supervisor` | `backend/src/app/gql/definitions/supervisor.graphql` | `backend/src/app/gql/schemas/SupervisorSchema.ts` | Supervisor actor reads |

Shared SDL:
- `backend/src/app/gql/definitions/base.graphql` — scalars, `_Ability`, `_Notification`, `_Timestamps`, `_Pagination`, `_OrganizationStatus` / `_OrganizationStatusValue`, `_MessageTemplateChannel` / `_MessageTemplateChannelValue`, `_MeetingType` / `_MeetingStatus` / `_MeetingNotifyStatus`, `_MeetingParticipantType` / `_MeetingParticipantDeliveryStatus` (+ Value enums)

On-disk draft SDL (reference copy):
- `backend/src/app/gql/definitions/shared.graphql` — notification query sketch; role SDL files are authoritative in codegen.

## Customer schema surface

Root queries (from `customer.graphql`):
- `me` — current customer profile
- `notifications` — paginated notifications
- `organization` — current customer's organization (`prepareOneGQLModel({ me: true })`)
- `members` — members of the customer's organization (`{ me: true }` → Organization parent)
- `member(id)` — single member in the customer's organization
- `messageTemplates` — message templates of the customer's organization (same `me` → Organization parent)
- `messageTemplate(id)` — single message template in the customer's organization
- `meetings` — meetings of the customer's organization (same `me` → Organization parent)
- `meeting(id)` — single meeting in the customer's organization

Nested (cardinality-safe):
- `_Member.organization: _Organization` (`belongsTo`, expected count 1)
- `_MessageTemplate.organization: _Organization` (`belongsTo`, expected count 1)
- `_Meeting.organization: _Organization`, `_Meeting.chairperson: _Member`, `_Meeting.whatsappTemplate` / `emailTemplate: _MessageTemplate`
- `_Meeting.participants: [_MeetingParticipant]` (roster; expected board size ≤ 100 — B15); `_MeetingParticipant.member: _Member`
- `_Meeting.agendaItems: [_AgendaItem]` (agenda lines; nested only — see `agenda-item-domain.md`)
- not `_Organization.members` / meetings / templates as nested high-cardinality lists (root list only)
- not root `meetingParticipants` / `meetingParticipant` (nested under meeting only)
- not root `agendaItems` / `agendaItem` (nested under meeting only)
- not `_Member.meetingParticipants` / `_Member.meetings` (deferred)

Bridges (`backend/src/app/gql/schemas/CustomerSchema.ts`):
- `MeBridge`
- `NotificationBridge`
- `OrganizationBridge`
- `MemberBridge` (extends `CustomerOrganizationOwnedBridgeBase`)
- `MessageTemplateBridge` (extends `CustomerOrganizationOwnedBridgeBase`)
- `MeetingBridge` (extends `CustomerOrganizationOwnedBridgeBase`)
- `MeetingParticipantBridge` (nested under Meeting only; extends `CustomerBridgeBase`)
- `AgendaItemBridge` (nested under Meeting only; extends `CustomerBridgeBase`)

Org-owned intermediate base: `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts`.

## Supervisor schema surface

Root queries (from `supervisor.graphql`):
- `me` — current supervisor profile
- `notifications` — paginated notifications
- `customers` — customer list
- `customer(id)` — single customer
- `customerStats` — aggregate stats
- `organizations(filter)` — organization list
- `organization(id)` — single organization

Nested relation:
- `_Customer.organization: _Organization` (requires `OrganizationBridge` registered)

Bridges (`backend/src/app/gql/schemas/SupervisorSchema.ts`):
- `MeBridge`
- `NotificationBridge`
- `CustomerBridge`
- `CustomerStatsBridge`
- `OrganizationBridge`

## Generated types

Codegen output under `backend/src/app/gql/gql-types/`:
- `customer.ts`
- `supervisor.ts`
- `base.ts`

## Frontend mirrors

| Frontend | Mirrors | Config | Current status |
|---|---|---|---|
| `website/` | `base` + `customer` | `website/graphql.config.yml` | Active |
| `cpanel/` | `base` + `supervisor` | `cpanel/graphql.config.yml` | **Deferred** — `cpanel/` checkout temporarily removed; do not sync supervisor mirrors until restored |

Sync (when the target platform exists) is command-based copy from backend SDL/types. Do not hand-edit mirrored files.

## Depth and security

- Bridge depth guards (`maxLevel`, `deepInclude`) remain active.
- Role isolation: customer schema never exposes supervisor-only fields.
- List queries must be bounded and ordered.

See `docs/platforms/backend/patterns/graphql-and-bridges.md` for authoring standards.

Organization domain detail: `docs/platforms/backend/contracts/organization-domain.md`.  
Member domain detail: `docs/platforms/backend/contracts/member-domain.md`.  
Message template domain detail: `docs/platforms/backend/contracts/message-template-domain.md`.  
Meeting domain detail: `docs/platforms/backend/contracts/meeting-domain.md`.
