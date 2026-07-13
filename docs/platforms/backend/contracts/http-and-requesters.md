# HTTP and Requesters Contract (Ejtmaa)

## HTTP mounts

Configured in `backend/src/resources/configs/http/express.ts`.

| Mount | Status | Actor granting | Route module |
|---|---|---|---|
| `/website` | Active | visitor, customer | `backend/src/app/http/routes/website.ts` |
| `/cpanel` | Active | visitor (login), supervisor | `backend/src/app/http/routes/cpanel.ts` |

Client portals are SSR frontends on `/website` and `/cpanel`.

## Middleware groups

- `website` — CORS, compression, json, query_values, granting (customer), local, errors_funnel
- `cpanel` — CORS, compression, json, query_values, granting (visitor for login, supervisor for authed), local, errors_funnel

## Requester dispatch

Controllers use `ControllerBase.requesterHandle(platform, actor)`.

Website controllers:
- `backend/src/app/http/controllers/website/forms/VisitorRequesterController.ts` — visitor auth
- `backend/src/app/http/controllers/website/forms/customer/CustomerRequesterController.ts` — customer writes
- `backend/src/app/http/controllers/website/data_adapters/GQLAdapterController.ts` — visitor GQL reads
- `backend/src/app/http/controllers/website/data_adapters/customer/GQLAdapterController.ts` — customer GQL reads

Cpanel controllers:
- `backend/src/app/http/controllers/cpanel/forms/SupervisorRequesterController.ts` — visitor login and supervisor writes
- `backend/src/app/http/controllers/cpanel/data_adapters/GQLAdapterController.ts` — supervisor GQL reads

## Active requesters (6)

| Requester | `@requester` ident | Website subs | Cpanel subs |
|---|---|---|---|
| AuthRequester | `auth` | visitor | visitor |
| CustomerRequester | `customer` | customer | supervisor |
| NotificationRequester | `notification` | customer | — |
| SupervisorRequester | `supervisor` | — | supervisor |
| WebsiteSettingsRequester | `website_settings` | — | supervisor |
| PlatformSettingsRequester | `platform_settings` | — | supervisor |

Source files: `backend/src/app/orchestrator/requesters/`
Registration: `backend/requesters.website.ts`, `backend/requesters.cpanel.ts`

## Custom start

SSR boot hydration endpoints:

| Path | Response auth key | Shape |
|---|---|---|
| `/website/custom/start` | `auth` | `{ token?, authedAs?, user? }` |
| `/cpanel/custom/start` | `auth` | `{ token?, authedAs: "SUPERVISOR"?, user? }` |

Both mounts return `auth` (not `authentication`). See `backend/src/app/http/controllers/website/custom/StartController.ts` and `backend/src/app/http/controllers/cpanel/custom/StartController.ts`.

See `docs/platforms/backend/contracts/client-portal-http-website.md` and `docs/platforms/cpanel/login-runtime-and-feedback.md`.
