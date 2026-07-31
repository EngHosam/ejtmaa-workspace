# Runtime and Integrations

Interpretation rule:
- sections below describe active Ejtmaa backend providers and integration contracts for implementers.

## 1) Application Runtime Composition

Application boot starts at `backend/src/index.ts` and initializes `App` with `backend/src/resources/configs/app.ts`.

Active providers:
- logger
- localization
- orm
- http
- socket
- notify
- render
- validation
- scheduler
- mailer
- smart-worker
- gql
- orchestrator

## 2) Environment and Configuration

Primary environment contract is defined in `backend/.env.example`.

Main groups:
- app/runtime (`NODE_ENV`, `DEV_MODE`, paths, timezone)
- http (`EXPRESS_PORT`, `EXPRESS_BASE_URL`, install mode ports)
- db (`DB_*`)
- socket (`IO_PORT`)
- organization host testing (`TEST_ORGANIZATION_ID`) — non-production organization resolved by `/website/custom/org/start`
- payment (`PAYMENT_MODE`, MyFatoorah keys)
- LiveKit media (`LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`) — see `docs/platforms/backend/contracts/livekit-media-plane.md`

## 3) HTTP Integration Details

Express driver is configured in `backend/src/resources/configs/http/express.ts`:
- Twig views enabled (`backend/src/resources/views`)
- static files served from `static/`
- multipart upload middleware (`multer`, temp storage at `storage/temporary`)
- route-scoped CORS policy:
  - allows Ejtmaa website origins for website routes
  - allows cpanel origin for cpanel routes
  - full allow in `DEV_MODE=true`

## 4) Localization and Validation

Localization config:
- `backend/src/resources/configs/local.ts`
- language files under `backend/src/resources/trans/{en,ar}`

Validation config:
- `backend/src/resources/configs/validation.ts`
- project-specific rules (`mobile`, `multi-lang`, `file`, `array`, `url`, and others)
- requester facts validators in `backend/src/app/validation/joi_rules.ts`

## 5) Notifications and Realtime

Notify provider config:
- `backend/src/resources/configs/notify/index.ts`
- broadcasters: socket.io + FCM
- events: `OnUserEvent`, `OnCustomerEvent`

Mirror contract:
- customer user-facing backend socket events must stay mirrored in `website/src/types/events.ts` and `website/src/resources/configs/socket/events.ts`
- supervisor user-facing backend socket events must stay mirrored in `cpanel/src/types/events.ts` and `cpanel/src/resources/configs/socket/events.ts`
- full contract: `docs/platforms/backend/contracts/socket-event-mirroring.md`

Socket provider config:
- `backend/src/resources/configs/socket/io.ts`
- namespaces: `/customer`, `/supervisor`, `/meeting`
- bootable middlewares: `AuthenticationIOMiddleware` (`auth`), `MeetingAuthenticationIOMiddleware` (`meeting_auth`)
- connection controllers: `ConnectionIOController` (`/customer`, `/supervisor`), `MeetingConnectionIOController` (`/meeting`)

Framework reference for the underlying package (`@nodejs/socket` drivers, route grammar, handler-array listener contract, execution queue, known limitations): `docs/platforms/backend/modules/nodejs-socket-library.md`.

`AuthenticationIOMiddleware` serves the actor namespaces (`/customer`, `/supervisor`) on a handshake `token` and throws `NOT_VALID_CREDENTIAL` when it does not resolve to a customer or supervisor.

`/meeting` uses `meeting_auth`: handshake requires `memberId` + `memberToken` (`Member.access_token`) + `meetingId`, proves roster membership and ACTIVE organization, optionally checks `organizationId` against the meeting tenant, and stores `MeetingSocketData`. `MeetingConnectionIOController` joins `Rooms.MEETING(meetingId)` and binds the live event set.

`/meeting` child events (collaborative session):

| Event | Controller alias | Class | Behavior |
|---|---|---|---|
| `meeting.live.sync` | `meeting_live_sync` | `controllers/meeting/MeetingLiveSyncIOController` | answers the caller with the missing document diff + the server state vector |
| `meeting.live.update` | `meeting_live_update` | `controllers/meeting/MeetingLiveUpdateIOController` | validates + status-gates the update, applies it, broadcasts to the room |

Both are **client → server** inbound events; the server replies on the same names plus `meeting.live.error` for a rejection. None of them are notify events, and none are mirrored into `website`/`cpanel` event registries (`socket-event-mirroring.md` does not apply). Namespace contract: `docs/platforms/backend/contracts/meeting-realtime-socket.md`. State plane (Yjs document + `live_state` BLOB): `docs/platforms/backend/contracts/meeting-live-state.md`. Website consumer (transport + linking gate): `docs/platforms/website/organization-host-routing.md` §5.1, §5.3.

Room conventions are centralized in:
- `backend/src/resources/consts/NotificationsConsts.ts` — `Namespaces.MEETING` (`/meeting`), `FCM_Namespaces.MEETING` (`-meeting`), `Rooms.MEETING(meetingId)`

Organization-host contract (website boot, HTTP start, gating, Meeting socket): `docs/platforms/website/organization-host-routing.md`.

## 6) Mailer and Templates

Mailer config:
- `backend/src/resources/configs/mailer.ts`
- SMTP host defaults to private email server
- `MainEmail` is the active email class

Template sources:
- email templates: `backend/src/resources/emails/*.twig`
- rendered page templates: `backend/src/resources/views/*.twig`

## 7) Payment and Invoice Helpers

Active helpers:
- `backend/src/app/helpers/MyFatoorah.ts`
- `backend/src/app/helpers/PdfInvoiceHelper.ts`
- `backend/src/app/helpers/QRInvoiceHelper.ts`

Current responsibilities:
- payment method listing and payment URL creation
- supplier/bank operations through MyFatoorah API
- PDF invoice rendering via Twig + Puppeteer
- QR payload generation for VAT-compliant invoice encoding

Related templates:
- `backend/src/resources/views/invoice.twig`
- `backend/src/resources/views/my_fatoorah_done.twig`

## 7b) LiveKit media plane

Active helper:
- `backend/src/app/helpers/LiveKitHelper.ts`

Join-token HTTP (organization host):
- `POST /website/custom/org/livekit_token` + per-route `org_host`
- Controller: `backend/src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts`
- Response: `{ token, url }` (JWT + client connect URL)
- Website: `useMeetingLiveKitToken` + `API.CUSTOM.ORG_LIVEKIT_TOKEN` → `useMeetingLiveKitRoom` (`Room.connect`) → `MeetingLiveBroadcast`

Dependency: `livekit-server-sdk@2.17.0` (pinned) in `backend/`; browser client `livekit-client` is pinned in `website/` only.

Helper responsibilities:
- derive room name `ejtmaa:meeting:{meetingId}`
- create/list/delete rooms and mint per-participant access tokens
- normalize client connect URL (`ws`/`wss`)

Controller owns roster/org/`STARTED` (peek) authz and participant JWT cache writes. Helper does **not** own those, attendance SQL, or website `Room.connect`. Full contract: `docs/platforms/backend/contracts/livekit-media-plane.md`; browser side: `docs/platforms/website/flow-meeting-broadcast.md`.

## 8) Database Scripts and Patches

SQL scripts:
- `backend/src/resources/scripts/routines.sql` (intentionally comments-only placeholder)
- `backend/src/resources/scripts/routines_refresh.sql`

Patches:
- `SeedPatch` for baseline settings, supervisor bootstrap, and demo-customer hook

`SeedPatch.init()` seeds:
- system settings defaults (about, terms, privacy, seller_name = "Ejtmaa", VAT flags)
- default supervisor admin account when none exists
- demo customers via `seedDemoCustomers` (extension hook for demo rows)

`SeedPatch.update(version)` is reserved for future targeted maintenance actions. The checked-in switch currently rejects unknown versions with `NOT_VALID`.

## 9) Scheduler and Smart Worker

Scheduler:
- provider config in `backend/src/resources/configs/scheduler`
- sample task class `backend/src/app/scheduler/tasks/TestTask.ts` registered in `cron.ts` for framework wiring
- production domain cron jobs land as modules are added

Smart worker:
- config in `backend/src/resources/configs/smart-worker.ts`
- sample nano worker in `backend/src/smart-worker/workers/nano`

## 10) Operational Notes

- `CONSOLE_ENABLED=true` starts interactive console commands at app boot.
- Console root: `backend/src/console/Console.ts`.
- Keep external side effects behind explicit provider usage and post-commit boundaries in requester flows.
