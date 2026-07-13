# CPanel Data Flow, Forms, and GQL Foundation

## Purpose

This document defines the data-flow architecture for `cpanel/`.
It maps the shared SSR adapter/form mental model onto the supervisor control-panel stack.

Scope:
- control panel bootstrap differs from `website/`,
- data/form organization follows the same adapter, form, injector, and store concepts.

Shared shell UI (`MainLayout`, `Drawer`, `Footer`, `UiMockup`, translations, shell assets) lives in the layout/component layer and does not change the read/write ownership model here.

## 1) Shared mental model

The cpanel repository includes:
- store-owned `data_adapters`,
- store-owned `forms`,
- adapter hooks,
- form hooks,
- shallow injector hooks,
- frontend endpoint mapping for `/cpanel/data_adapters/gql`,
- frontend endpoint mapping for `/cpanel/forms/requester/:requester/:sub`.

That means the intended architecture is already visible:

- **read data** should flow through adapters,
- **write actions** should flow through forms/requesters,
- **route screens** should orchestrate those boundaries instead of bypassing them with ad-hoc axios logic.

## 2) Reads belong to adapters

Primary local evidence:
- `cpanel/src/resources/configs/store/data-adapters.ts`
- `cpanel/src/resources/configs/store.ts`
- `cpanel/src/app/ui/base/components/Adapter.tsx`
- `cpanel/src/app/ui/base/hooks/useAdapter.tsx`
- `cpanel/src/app/ui/base/hooks/useShallowAdapter.tsx`
- `cpanel/src/resources/configs/axios/api.ts`

### Rule

When a screen needs backend-owned read data, prefer the adapter layer.

The adapter boundary should own:
- API endpoint,
- query payload,
- records/section data,
- status,
- refresh/load lifecycle,
- pagination identity when relevant.

The page/screen should own:
- layout and rendering,
- mapping adapter state into page UI names,
- deciding when a refresh should happen,
- composing the visual result.

### Why this matters

This keeps cpanel aligned with the same high-value discipline already used in `website/`:
- data-loading state has one owner,
- refresh behavior has one owner,
- route UI does not become transport code.

## 3) Writes belong to forms/requesters

Primary local evidence:
- `cpanel/src/resources/configs/store/forms.ts`
- `cpanel/src/resources/configs/store.ts`
- `cpanel/src/app/ui/base/components/Form.tsx`
- `cpanel/src/app/ui/base/hooks/useForm.tsx`
- `cpanel/src/app/ui/base/hooks/useShallowForm.tsx`
- `cpanel/src/resources/configs/axios/api.ts`

### Rule

When a screen performs a requester-backed write action, prefer the form layer.

The form boundary should own:
- requester endpoint configuration,
- values,
- errors,
- send/reset lifecycle,
- submit status.

The page/component should own:
- visual field composition,
- choosing the correct `sub`,
- guarding duplicate submits,
- deciding post-success navigation or read refresh behavior.

### Implication

Control-panel mutations should not default to raw `axios.post(...)` inside pages when the form/requester boundary is the true architecture for that interaction.
Supervisor GQL mirrors live under:

- `cpanel/src/app/ui/pages/Login.tsx` exercises a requester-backed shallow form,
- the page initializes the form with `API.FORMS.R("auth")`,
- and it sends sub `supervisorLogin` through the existing cpanel visitor requester contract.

## 4) `useAdapter` versus `useShallowAdapter`

Cpanel scaffold already mirrors the shared split:

### `useAdapter`

Use it when the adapter already exists in store state under a stable identity.

Good fit:
- real shared adapter boundaries registered in the store,
- stable singleton-like reads reused across screens.

### `useShallowAdapter`

Use it when the screen must inject its own adapter instance first.

Good fit:
- route-scoped detail reads,
- filtered list screens,
- domain pages that need deterministic route-based identities,
- screens with multiple local adapters.

### Important local implementation detail

`useShallowAdapter` already merges:
1. inherited adapter init props,
2. injector init props,
3. direct init props.

This is the same structural benefit seen in `website/`:
- inherit shared endpoint defaults,
- override only what the screen owns.

## 5) `useForm` versus `useShallowForm`

The same split exists for forms.

### `useForm`

Use it for a truly existing shared form identity.

Good fit:
- stable shared forms,
- reused auth-like forms once those become real project boundaries.

### `useShallowForm`

Use it for route-, modal-, or page-scoped form boundaries that must be injected at runtime.

Good fit:
- per-screen create/update forms,
- temporary edit flows,
- page-local but still store-owned requester forms.
- the login page in `cpanel`, which uses a route-scoped form identity while still respecting the shared forms/requester architecture.

### Important local implementation detail

`useShallowForm` already merges:
1. inherited form init props,
2. injector init props,
3. direct init props.

This is the correct path for future feature forms that inherit a known requester endpoint but still need page-specific initial values or behavior.

## 6) `DATA_ADAPTERS` and `Forms` are not dumping grounds

Shared store identifiers (`ADAPTER1`, `FORM1`) are starter placeholders in:
- `cpanel/src/resources/configs/store/data-adapters.ts`
- `cpanel/src/resources/configs/store/forms.ts`

### Rule

Shared store identifiers should be added only for **real shared boundaries**.

Good future candidates:
- supervisor profile/me,
- globally reused home stat summaries that truly cross multiple routes,
- stable auth/login form,
- widely reused settings forms.

Avoid:
- creating one global identifier for every temporary route,
- promoting screen-private boundaries into the shared registry without real reuse value.

When a boundary is route-scoped, prefer deterministic `useShallowAdapter` / `useShallowForm` identities.

## 7) GQL is the intended read contract

Evidence:
- `cpanel/src/resources/configs/axios/api.ts` exposes `/data_adapters/gql`
- `cpanel/graphql.config.yml` already reserves base + supervisor schema inputs
- `cpanel/src/types/gql/definitions/{base,supervisor}.graphql` — local SDL mirror
- `cpanel/src/types/gql/gql-types/{base,supervisor}.ts` — local generated type mirror
- `cpanel/src/app/ui/pages/graphql.config.yml` and `cpanel/src/app/ui/components/graphql.config.yml` scope local UI tooling to the mirrored schema pair
- the repo already depends on `graphql` and `gql`

### Rule

Control-panel read integration is centered on GQL through the adapter layer.

That means:
- use the data adapter boundary for GraphQL reads,
- keep route screens consuming adapter state rather than issuing raw page-local GraphQL transport calls,
- align frontend query usage with the supervisor backend schema,
- keep requester actions for writes and command-like operations.

### GQL read surfaces

Supervisor GQL-backed read flows in cpanel:

- customer list and detail (`customers`, `customer`)
- home stat KPI (`customerStats.total_count`)
- notifications (`notifications`)
- supervisor profile (`me`)

Mirrored SDL and types live under `cpanel/src/types/gql/**` (`base` + `supervisor`).

Reads use `DATA_ADAPTERS.GQL`. Writes use supervisor requester forms (`auth`, `supervisor`, `customer`, `website_settings`, `platform_settings`).

### Mirror/tooling discipline

Mirror discipline:

- backend GraphQL files remain the source of truth,
- cpanel mirrors the relevant SDL and generated `gql-types` under `src/types/gql/`,
- supervisor mirrors belong to `cpanel/` only and must not be copied into `website/` unless that client truly consumes the supervisor role surface,
- local `graphql.config.yml` files belong only in UI subtrees that actually host GraphQL-aware code,
- helper configs scope tooling only and document GraphQL editor boundaries, not runtime data features by themselves.

## 8) Relationship to backend requesters

Evidence:
- `cpanel/src/types/requesters/requesters.cpanel.ts`
- `backend/requesters.cpanel.ts`
- `docs/platforms/backend/contracts/http-and-requesters.md`

### Rule

The frontend requester type map must stay aligned with backend reality.

State:
- frontend requester map exists structurally,
- visitor login contract: `auth.supervisorLogin`,
- supervisor requester subs: `supervisor.read`, `supervisor.update`,
- customer requester subs: `customer.read`, `customer.update`,
- settings requesters: `website_settings`, `platform_settings`.

Future implementation must:
- add truthful requester subs on the frontend,
- keep names and scope aligned with backend `/cpanel` requesters,
- avoid inventing frontend-only requester names that do not map to backend ownership.

## 9) Screen responsibility split

A future cpanel page should follow this structure:

1. page identifies route ownership,
2. page selects adapter/form hooks,
3. page maps state into UI-facing names,
4. shared components render the interface,
5. adapters own read state,
6. forms own write state,
7. services own router/auth/global coordination.

This mirrors the strongest lesson from `website/`:
- pages orchestrate,
- hooks and boundaries own state,
- components own visual reuse,
- backend contracts remain explicit.

Route-level evidence:

- `Login` — form/requester boundary for supervisor login,
- `Customers` — route-state-backed GQL reads through shallow adapters,
- `Customer` — route-scoped form reads/writes,
- `Home` — `HomeStatCard` from `customerStats.total_count`,
- `AccountSettings` — supervisor password change,
- `Error` — route-owned presentational fallback.

## 10) Shared usability expectations from `website/`

Although cpanel is a web platform, the same interaction quality should be preserved when the concepts overlap:

- loading is explicit,
- empty is explicit,
- error is explicit,
- busy state guards duplicate actions,
- refresh behavior is deterministic,
- route-owned screens do not mutate unseen shared state accidentally,
- one mutation refreshes only the affected read boundaries.

These expectations apply to supervisor routes: paginated/filterable tables, home stat surfaces (`customerStats.total_count`), settings forms, and role/permission-dependent pages.

## 12) Final rule

For every new control-panel screen, ask:

```text
Is this backend-owned read data?
  Use an adapter, ideally through the GQL path.

Is this backend-owned write action?
  Use a form/requester boundary.

Is this route-specific state?
  Prefer useShallowAdapter/useShallowForm with a deterministic identity.

Is this truly shared across screens?
  Promote it into DATA_ADAPTERS or Forms only then.
```

## Related

- `docs/platforms/cpanel/component-structure.md`
- `docs/platforms/cpanel/ui-foundation.md`
- `docs/platforms/cpanel/overview.md`
- `docs/platforms/backend/contracts/http-and-requesters.md`
