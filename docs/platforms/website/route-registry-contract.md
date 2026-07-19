# Website Route Registry Contract

Customer portal contract (see `overview.md`).

Authoritative contract for `website/src/resources/configs/routes.ts` and the mirrored `MPagesRoutes` interface in the same file.

## 1) Purpose

- Single routing authority for the SSR Customer website (`@my-ssr/web-core` first-match router).
- Namespace authed workspace URLs under `/customer/*`.
- Keep `routes` object keys and `MPagesRoutes` members grouped and ordered by audience: **public → customer → base**.

Related runtime: `website/src/app/services/router.ts` (`publicRoutes`, `applyRouterMiddleware`, `getMyHomeIdentify`).

## 1.1) Shipped state

`routes.ts` currently ships **customer** identifies with `customerRouter` including `CustomerHome`, `CustomerMembers`, multi-path `CustomerMemberForm`, `CustomerMeetings`, `CustomerMeetingForm`, `CustomerMeetingDetails`, `CustomerOrganization`, `CustomerMessageChannels`, and multi-path `CustomerMessageChannelForm` on `CUSTOMER_MAIN` (public → customer → base section split). Remaining `/customer/*` workspace routes are still planned (see §5.2).

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Register` | `/register` | `BASIC` |
| `ResetPassword` | `/reset-password` | `BASIC` |
| `Home` | `/` | `LANDING` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `CustomerHome` | `/customer` | `CUSTOMER_MAIN` |
| `CustomerMembers` | `/customer/members` | `CUSTOMER_MAIN` (breadcrumb → `CustomerHome`) |
| `CustomerMemberForm` | create `/customer/members/form`; update `/customer/members/form/:id` | `CUSTOMER_MAIN` (breadcrumb → `CustomerMembers`) |
| `Error` | `/:error(404|500|403)` | `BASIC` (last entry) |

`MPagesRoutes` mirrors the same identifies; `CustomerMemberForm: { id?: string }` (`id` present ⇒ update). `publicRoutes = ["Login", "Register", "ResetPassword", "UiMockup", "Home"]`. `getMyHomeIdentify` returns `"CustomerHome"` when `authedAs === "CUSTOMER"`, else `"Home"`. Layouts shipped: `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`), `CustomerMainLayout` (`CUSTOMER_MAIN`).

## 2) Path helper (mandatory for workspace routes)

Declared once at the top of `routes.ts` (after re-exports, before `const routes`):

```ts
const customerRouter = (path: string) => `/customer${path}`;
```

| Helper | Use |
|--------|-----|
| `customerRouter("")` | Customer role home → `/customer` |
| `customerRouter("/settings")` | Customer feature paths → `/customer/settings`, … |

**Rules:**

1. Every `mustAuthedAs: ["CUSTOMER"]` route path MUST go through `customerRouter(...)` (including multi-path form routes' `create` / `update` strings).
2. **Public** routes (`Login`, `Home`, `Register`, `ResetPassword`, `UiMockup`) and **`Error`** keep absolute paths — do **not** wrap them in the helper.
3. Do **not** hand-write `/customer/...` literals on workspace routes when the helper applies — one edit surface prevents drift.

Navigation in app code SHOULD use `nav.push({ identify, params?, query? })` (typed `PagesRoutes`), not hard-coded path strings. Exceptions: external redirects, share URLs, and third-party callbacks — those MUST match the registered paths in this document.

## 3) Registry section blocks (mandatory)

Both `const routes: RouterRoutes` and `export interface MPagesRoutes` use the same comment blocks, in this order:

| Section | Route keys |
|---------|------------|
| **public** | `Login`, `Home`, `Register`, `ResetPassword`, `UiMockup` |
| **customer** | `CustomerHome` and all customer `mustAuthedAs` routes |
| **base** | `Error` (routes only; `MPagesRoutes` has an empty base section — `Error` has no typed nav params) |

When adding a route:

1. Place the `routes` entry in the correct section block.
2. Add the matching `MPagesRoutes` member in the **same** section block (same relative order within the section).
3. Keep **identify key order** stable inside each section.

## 4) Registration order and first-match semantics

`web-core` resolves the first matching route while iterating `routes` object key order. Therefore:

1. **Public block MUST stay first** — visitor/auth paths must not be registered after workspace routes they could collide with.
2. **Customer block before `Error`.**
3. **`Error` MUST stay last** in the `base` section (`//in last` comment preserved).
4. **Fixed segments before `:param` on the same prefix** — see §4.1.

### 4.1) Fixed segment before parametric (`:id`)

When a fixed URL segment shares a prefix with a parametric route, the **fixed route(s) MUST appear earlier in `routes` object key order** than the parametric route.

**Example (static-before-parametric on settings forms):**

1. Fixed-segment routes register before parametric siblings on the same prefix per `website-route-static-before-parametric.mdc`.

**Example (meetings):** `CustomerMeetingForm` (`/customer/meetings/form`) is registered **before** `CustomerMeetingDetails` (`/customer/meetings/:id`) in `routes.ts`.

Rule: `.cursor/rules/website-route-static-before-parametric.mdc`. Skill: `.cursor/skills/website-route-static-before-parametric/SKILL.md`. Invariant: `docs/invariants/website.md` W41.

### 5.1) Public (no auth)

Shipped: `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`.

| Identify | Path | Layout |
|----------|------|--------|
| `Login` | `/login` | `BASIC` |
| `Register` | `/register` | `BASIC` |
| `ResetPassword` | `/reset-password` | `BASIC` |
| `Home` | `/` | `LANDING` |
| `UiMockup` | `/ui-mockup` | `MAIN` |

### 5.2) Customer (`mustAuthedAs: ["CUSTOMER"]`)

Shipped: `CustomerHome`, `CustomerMembers`, `CustomerMemberForm`, `CustomerMeetings`, `CustomerMeetingForm`, `CustomerMeetingDetails`, `CustomerOrganization`, `CustomerMessageChannels`, `CustomerMessageChannelForm`, `CustomerMessageTemplates`, `CustomerMessageTemplateForm`. Remaining workspace routes are planned.

| Identify | Path | Status |
|----------|------|--------|
| `CustomerHome` | `/customer` | shipped — command map (`CUSTOMER_MAIN`); no breadcrumb — `flow-customer-shell.md` §7 |
| `CustomerMeetings` | `/customer/meetings` | shipped — directory (ResultLane + search + status); drawer tile; `breadcrumb: { parent: CustomerHome }` — `flow-customer-meetings.md` |
| `CustomerMeetingForm` | `/customer/meetings/form` | shipped — create form (`Forms.CUSTOMER_MEETING`); before `:id`; `breadcrumb: { parent: CustomerMeetings }` — `flow-customer-meetings.md` |
| `CustomerMeetingDetails` | `/customer/meetings/:id` | shipped — preparation and approval workspace; `breadcrumb: { parent: CustomerMeetings }` — `flow-customer-meetings.md` |
| `CustomerMembers` | `/customer/members` | shipped — directory (ResultLane + search); drawer tile; `breadcrumb: { parent: CustomerHome }` — `flow-customer-members.md` |
| `CustomerMemberForm` | `/customer/members/form` (+ `/:id`) | shipped — multi-path create/update; `breadcrumb: { parent: CustomerMembers }`; href via `formRoute.ts` — `flow-customer-members.md` §5 |
| `CustomerOrganization` | `/customer/organization` | shipped — settings form upsert; drawer tile; `breadcrumb: { parent: CustomerHome }` — `flow-customer-organization.md` |
| `CustomerMessageChannels` | `/customer/message-channels` | shipped — directory (ResultLane); drawer tile before Settings; `breadcrumb: { parent: CustomerHome }` — `flow-customer-message-channels.md` |
| `CustomerMessageChannelForm` | `/customer/message-channels/form` (+ `/:id`) | shipped — multi-path create/update; parent `CustomerMessageChannels` — `flow-customer-message-channels.md` |
| `CustomerMessageTemplates` | `/customer/message-templates` | shipped — `flow-customer-message-templates.md` |
| `CustomerMessageTemplateForm` | `/customer/message-templates/form` (+ `/:id`) | shipped — multi-path form |
| `CustomerSubscription` | `/customer/subscription` (target) | planned — drawer tile |
| `CustomerSettings` | `/customer/settings` | planned — drawer tile + settings flow |
| `CustomerSupport` | `/customer/support` (target) | planned — drawer tile (no GQL root yet) |
| `CustomerHelpGuide` | `/customer/help-guide` | planned — drawer tile + static info |
| `CustomerNotifications` | `/customer/notifications` | planned — header bell |
| `CustomerAbout` | `/customer/about` | planned — static info |
| `CustomerTerms` | `/customer/terms` | planned — static info |

Drawer tile identifies and gating: `flow-customer-shell.md` §5.3. Paths marked `(target)` are not registered until their page stage; drawer shows them disabled until `routes` membership exists.

### 5.3) Base

| Identify | Path | Layout |
|----------|------|--------|
| `Error` | `/:error(404|500|403)` | `BASIC` |

## 6) Role redirect alignment

Shipped:

- `getMyHomeIdentify` returns `CustomerHome` when `authedAs === "CUSTOMER"`, else `Home`.
- Authed users on public routes redirect to `getMyHomeIdentify` (→ `CustomerHome` for customer).
- Unauthed users on `/customer/*` redirect to `Login` (current middleware).

See `flow-auth.md` §4A and `.cursor/rules/website-auth-flow.mdc`.

## 7) Traceability

| Path | Repo | Change |
|------|------|--------|
| `website/src/resources/configs/routes.ts` | website | `customerRouter`; public/customer/base; `CustomerHome` + `CustomerMembers` + `CustomerMemberForm` multi-path |
| `website/src/resources/configs/customer/formRoute.ts` | website | `buildCustomerMemberFormHref` |
| `website/src/types/extends/global.ts` | website | `BreadcrumbMeta` on `PageRouteState` |
| `website/src/app/services/router.ts` | website | `publicRoutes`, `getMyHomeIdentify` → `CustomerHome` for customer |
| `website/src/app/ui/layouts/CustomerMainLayout.tsx` | website | `CUSTOMER_MAIN` shell + sub-header offset |
| `website/src/app/ui/pages/customer/CustomerHome.tsx` | website | authed home command map — `flow-customer-shell.md` §7 |
| `website/src/app/ui/pages/customer/CustomerMembers.tsx` | website | members directory host + W38 breadcrumb (`flow-customer-members.md`) |
| `website/src/app/ui/pages/customer/CustomerMemberForm.tsx` | website | member form host + W38 breadcrumb |
| `docs/platforms/website/flow-customer-shell.md` | root | shell + drawer IA + breadcrumb / `HomeMark` |

## 8) Related documents and rules

- `docs/platforms/website/overview.md` §6.1 — summary pointer
- `docs/invariants/website.md` W22 (role redirect), **W37** (route registry)
- `.cursor/rules/website-route-registry-governance.mdc` — agent enforcement
- `.cursor/rules/website-auth-flow.mdc` — `publicRoutes` + middleware
- Per-feature flows under `docs/platforms/website/flow-*.md` — path columns reference this contract
- `docs/platforms/website/flow-customer-members.md` — shipped members directory + form
- `.cursor/rules/website-multi-path-form-routes.mdc` — create/update path object contract
