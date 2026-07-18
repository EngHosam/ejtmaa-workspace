# Backend Platform Overview (Ejtmaa)

Interpretation rule:
- sections below describe the active Ejtmaa backend contract for implementers.

## 1) Architectural Intent

Ejtmaa backend preserves the orchestration model for customer and supervisor administration.

Dominant execution model:
1. HTTP route or GraphQL adapter receives request.
2. Actor context is resolved (`visitor`, `customer`, `supervisor`).
3. Write-path executes through requesters.
4. Read-path executes through role GraphQL schema + bridges.
5. Models own persistence contracts and associations.
6. Side systems (notify, mailer, scheduler, console) run as providers.

Policy:
- Requesters + GraphQL remain the primary backbone for new features.

## 2) Runtime Composition

Entry:
- `backend/src/index.ts` boots `App` with `backend/src/resources/configs/app.ts`.

Provider graph:
- Core: logger, localization, validation, orm, http, socket.
- Feature infra: notify, mailer, scheduler, gql, smart-worker.
- Orchestration: `backend/src/app/orchestrator/Orchestrator.ts`.

## 3) Active Domain Surface

ORM models (`backend/src/app/orm/models/`):
- `Customer` — customer actor profile; `hasOne Organization` via `customer_id` (`getOrganization` / `createOrganization`)
- `Organization` — tenant entity owned by a customer (`customer_id`, one-to-one); `hasMany Member`, `hasMany MessageTemplate`, `hasMany Meeting`
- `Member` — non-actor org person (UUID `id` + `access_token`); belongs to Organization; ORM `hasMany` MeetingParticipant (no Member→meetings GQL yet)
- `MessageTemplate` — non-actor org message library (`WHATSAPP` | `EMAIL`); belongs to Organization
- `Meeting` — non-actor org session (UUID PK; chairperson Member; optional template FKs; LiveKit later); `hasMany` participants + agendaItems + decisions
- `MeetingParticipant` — non-actor roster join (composite PK `(meeting_id, member_id)`; type `CHAIRPERSON` | `MEMBER` | `VIEWER`)
- `AgendaItem` — non-actor agenda line under Meeting (`modelName: "agendaItem"`; durable SQL)
- `Decision` — non-actor decision under Meeting (`modelName: "decision"`; phase PRE_START|DURING; durable SQL)
- `Supervisor` — supervisor actor profile
- `User` — shared user identity
- `Token` — auth tokens
- `Notification` — in-app notifications
- `SystemSetting` — platform settings

## 4) Interface Layers

### HTTP

Configured in `backend/src/resources/configs/http/express.ts`:
- Route groups: `/website`, `/cpanel`
- Middleware groups: `website`, `cpanel`

Controllers autoload from `backend/src/app/http/controllers/website/` and `backend/src/app/http/controllers/cpanel/`.

### Requesters (6)

| Requester | Ident | Platforms |
|---|---|---|
| AuthRequester | `auth` | website (visitor), cpanel (visitor) |
| CustomerRequester | `customer` | website (customer), cpanel (supervisor) |
| NotificationRequester | `notification` | website (customer) |
| SupervisorRequester | `supervisor` | cpanel (supervisor) |
| WebsiteSettingsRequester | `website_settings` | cpanel (supervisor) |
| PlatformSettingsRequester | `platform_settings` | cpanel (supervisor) |

Registration maps:
- `backend/requesters.website.ts`
- `backend/requesters.cpanel.ts`

### GraphQL

Provider config: `backend/src/resources/configs/gql/index.ts`
- Schemas: `customer`, `supervisor`
- SDL: `backend/src/app/gql/definitions/customer.graphql`, `backend/src/app/gql/definitions/supervisor.graphql`, `backend/src/app/gql/definitions/base.graphql`
- Customer bridges: `MeBridge`, `NotificationBridge`, `OrganizationBridge`, `MemberBridge`, `MessageTemplateBridge`, `MeetingBridge`, `MeetingParticipantBridge`, `AgendaItemBridge`, `DecisionBridge` under `backend/src/app/gql/bridges/customer/` (org-owned children share `CustomerOrganizationOwnedBridgeBase`; participant/agenda/decision bridges are nested-only)
- Supervisor bridges: `MeBridge`, `NotificationBridge`, `CustomerBridge`, `CustomerStatsBridge`, `OrganizationBridge` under `backend/src/app/gql/bridges/supervisor/`

### Socket

Config: `backend/src/resources/configs/socket/io.ts`
- Namespaces: `/customer`, `/supervisor`
- Events: `OnUserEvent` (supervisor broadcast), `OnCustomerEvent` (per-customer)

## 5) Related docs

- `docs/platforms/backend/README.md` — contract index
- `docs/platforms/backend/contracts/http-and-requesters.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/invariants/backend.md`
