# Website Route Registry Contract

Customer portal contract (see `overview.md`).

Authoritative contract for `website/src/resources/configs/routes.ts` and the mirrored `MPagesRoutes` interface in the same file.

## 1) Purpose

- Single routing authority for the SSR Customer website (`@my-ssr/web-core` first-match router).
- Namespace authed workspace URLs under `/customer/*`.
- Keep `routes` object keys and `MPagesRoutes` members grouped and ordered by audience: **public → customer → base**.

Related runtime: `website/src/app/services/router.ts` (`publicRoutes`, `applyRouterMiddleware`, `getMyHomeIdentify`).

## 1.1) Shipped state

`routes.ts` currently ships **six** identifies. The `customerRouter` helper, the `/customer/*` workspace, `CustomerHome`, and the `public → customer → base` section split are **planned** (see §2–§6 for the target contract).

| Identify | Path | Layout |
|---|---|---|
| `Login` | `/login` | `BASIC` |
| `Register` | `/register` | `BASIC` |
| `ResetPassword` | `/reset-password` | `BASIC` |
| `Home` | `/` | `LANDING` |
| `UiMockup` | `/ui-mockup` | `MAIN` |
| `Error` | `/:error(404|500|403)` | `BASIC` (last entry) |

`MPagesRoutes` mirrors `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup` (no typed params). `publicRoutes = ["Login", "Register", "ResetPassword", "UiMockup", "Home"]`. `getMyHomeIdentify` returns `"Home"`. Layouts shipped: `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`). No `CUSTOMER_MAIN` layout yet.

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

### 5.2) Customer (`mustAuthedAs: ["CUSTOMER"]`) — planned

None of these routes ship yet. `customerRouter` and the `/customer/*` workspace are planned.

| Identify | Path |
|----------|------|
| `CustomerHome` | `/customer` |
| `CustomerNotifications` | `/customer/notifications` |
| `CustomerHelpGuide` | `/customer/help-guide` |
| `CustomerAbout` | `/customer/about` |
| `CustomerTerms` | `/customer/terms` |
| `CustomerSettings` | `/customer/settings` |

### 5.3) Base

| Identify | Path | Layout |
|----------|------|--------|
| `Error` | `/:error(404|500|403)` | `BASIC` |

## 6) Role redirect alignment — planned

Shipped: `getMyHomeIdentify` always returns `"Home"`; `publicRoutes` includes `Home`. The CUSTOMER-aware redirect logic below is planned.

- `getMyHomeIdentify` returns `CustomerHome` when `authedAs === "CUSTOMER"`, else `Home`.
- Authed users on public routes redirect to `CustomerHome`.
- Unauthed users on `/customer/*` redirect to `Home`.

See `flow-auth.md` §4A and `.cursor/rules/website-auth-flow.mdc`.

## 7) Traceability

| Path | Repo | Change |
|------|------|--------|
| `website/src/resources/configs/routes.ts` | website | `customerRouter`; public/customer/base sections |
| `website/src/app/services/router.ts` | website | `publicRoutes`, `getMyHomeIdentify`, middleware |

## 8) Related documents and rules

- `docs/platforms/website/overview.md` §6.1 — summary pointer
- `docs/invariants/website.md` W22 (role redirect), **W37** (route registry)
- `.cursor/rules/website-route-registry-governance.mdc` — agent enforcement
- `.cursor/rules/website-auth-flow.mdc` — `publicRoutes` + middleware
- Per-feature flows under `docs/platforms/website/flow-*.md` — path columns reference this contract
