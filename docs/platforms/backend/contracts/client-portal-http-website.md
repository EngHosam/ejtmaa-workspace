# Client Portal HTTP — Website (Ejtmaa)

## Scope

Ejtmaa customer client portal: the SSR frontend on backend mount **`/website`**.

## Mount

- Path: `/website`
- Route module: `backend/src/app/http/routes/website.ts`
- Middleware: `website` group (granting resolves visitor or customer)

## Consumer

- Repository: `website/`
- Axios base: configured in `website/src/resources/configs/axios/api.ts`
- Forms: `FORMS.R` (visitor), `FORMS.CUSTOMER.R`
- GQL adapter: `DATA_ADAPTERS.GQL` (visitor), `DATA_ADAPTERS.CUSTOMER.GQL` (customer)
- Customer GQL controller: `backend/src/app/http/controllers/website/data_adapters/customer/GQLAdapterController.ts`
- Custom start: `CUSTOM.START` → `/website/custom/start`
- Organization-host start: `CUSTOM.ORG_START` → `POST /website/custom/org/start`
- Organization-host LiveKit token: `CUSTOM.ORG_LIVEKIT_TOKEN` → `POST /website/custom/org/livekit_token`

## Start payload

`/website/custom/start` returns boot hydration including auth state under the `auth` key (SSR cookie-token model).

## Organization host start

`POST /website/custom/org/start` serves the organization-host boot of the same SSR portal (`{subdomain}.ejtmaa.live` or an organization custom domain).

- Route group: `OrgCustomRouter = CustomRouter("/org")()` in `backend/src/app/http/routes/website.ts` — **no** `auth` group middleware and **no** `org_host` middleware; the organization is resolved from the request body.
- Controller: `backend/src/app/http/controllers/website/custom/OrgStartController.ts`.
- Body (classified by the website, never re-derived from `Host`): `{ subdomain }` or `{ customDomain }`.
- Resolution: production matches `subdomain` / `custom_domain` case-insensitively and requires `status === "ACTIVE"`; a body with neither key throws `NOT_PERMIT`. Non-production ignores the body and resolves `TEST_ORGANIZATION_ID` only.
- Response: public branding only, under `organizationHost` — `id`, `name`, `description`, `logo_url`, `primary_color`, `secondary_color`. No auth, customer, or permission data.
- Unresolved organization → `404`.

Website contract: `docs/platforms/website/organization-host-routing.md`.

## `org_host` middleware

`backend/src/app/http/middlewares/OrganizationHostMiddleware.ts` is registered as `"org_host"` in `backend/src/resources/configs/http/middlewares/index.ts`. It reads the `organizationid` request header (the website sends `organizationId`; Express lowercases header names), loads the `ACTIVE` organization by id, throws `404` when missing or unresolved, and exposes `currentOrganization(req, sure?)`.

`/custom/org/start` intentionally does **not** use it (organization resolved from body). First wired consumer: `POST /custom/org/livekit_token` via per-route `middleware("org_host")` — see `livekit-media-plane.md` §6.

## Organization host LiveKit token

`POST /website/custom/org/livekit_token`:

- Middleware: `org_host` (per-route; not on the `OrgCustomRouter` group).
- Controller: `backend/src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts` (meeting-domain name under `custom/`).
- Body: `{ memberId, token, meetingId }` (`token` = `Member.access_token`).
- Response: `{ token, url }` — LiveKit JWT + client connect URL (`LiveKitHelper.clientUrl()`, `ws`/`wss`; recomputed from env, never persisted).
- Authz: member + meeting org + roster + in-process live-doc `STARTED` (`peekMeetingLiveDoc`); reuse-or-mint on `MeetingParticipant.livekit_*`.
- Errors: `NOT_VALID_CREDENTIAL`, `MEETING_NOT_LIVE`, `org_host` `404`.
- Website: `API.CUSTOM.ORG_LIVEKIT_TOKEN` + `useMeetingLiveKitToken` (union — `ready` carries `token` **and** `url`) → `useMeetingLiveKitRoom` connects the room for `MeetingLiveBroadcast`.

Full contract: `livekit-media-plane.md` §6. Website client: `../../website/flow-meeting-broadcast.md`.

## Actor model

| Actor | Access |
|---|---|
| visitor | Public routes, auth forms |
| customer | Authed customer routes and GQL |

## Cpanel counterpart

Supervisor portal: `/cpanel` consumed by `cpanel/` repository.
See `docs/platforms/cpanel/overview.md`.
