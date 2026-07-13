# Workspace Overview (Ejtmaa)

See also: `docs/README.md` (full documentation map).

## 1) Repository scope

The current workspace contains multiple coordinated repositories plus shared documentation:

- workspace root — shared documentation, Cursor rules/skills, and cross-platform governance
- `backend/` — Node/TypeScript backend runtime
- `cpanel/` — SSR control-panel frontend (Supervisor)
- `website/` — SSR customer web frontend

The workspace root itself is a git repository, and some platform folders are also their own git repositories.
Git, review, and history work must always be performed from the repository that actually owns the target files.

## 2) Platform status

Platform docs describe contracts for each repository. Checked-in trees implement those contracts.

### Backend

`backend/` is the server-side source of truth for:
- HTTP route mounts (`/website`, `/cpanel`)
- requesters (6 active: auth, customer, notification, supervisor, website_settings, platform_settings)
- GraphQL schemas (`customer`, `supervisor`)
- ORM models (Customer, Supervisor, User, Token, Notification, SystemSetting)
- integration providers (socket, notify, mailer, scheduler, console)

Detail:
- `docs/platforms/backend/README.md`
- `docs/platforms/backend/overview.md`
- `docs/invariants/backend.md`

### CPanel

`cpanel/` is the Supervisor SSR frontend contract on mount `/cpanel`.
See `docs/platforms/cpanel/overview.md` for repository layout and module catalog.

Detail:
- `docs/platforms/cpanel/README.md`
- `docs/platforms/cpanel/overview.md`
- `docs/invariants/cpanel.md`

### Website

`website/` is the Customer SSR frontend contract on mount `/website`.
Built on `@my-ssr/web-core` + `@typescript/sys-core` (same family as `cpanel/`).
See `docs/platforms/website/overview.md` Documentation stance.

Detail:
- `docs/platforms/website/README.md`
- `docs/platforms/website/overview.md`
- `docs/invariants/website.md`

## 3) Cross-platform contracts

- Customer writes/reads: `/website` + `customer` GQL schema
- Supervisor writes/reads: `/cpanel` + `supervisor` GQL schema
- Payment callbacks (target): `/external` + MyFatoorah finalize contract — `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md`
- Socket namespaces: `/customer`, `/supervisor`
- Active notify events:
  - `OnUserEvent` — supervisor broadcast; payload kind `NEW_CUSTOMER`
  - `OnCustomerEvent` — per-customer channel; payload kind `UPDATED`
- Socket mirror contract: `docs/platforms/backend/contracts/socket-event-mirroring.md`

## 4) Brand / design

Source of truth: `website/src/resources/configs/theme.ts`
- Documented in `docs/design-color-system.md` and `docs/design-system/colors.md`
- Shared by `website/` and `cpanel/` through `theme.ts` (when present in cpanel checkout)
