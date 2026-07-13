# CPanel Customer Management

Target supervisor cpanel contract (see `overview.md` and `supervisor-admin-modules.md`).

## 1) Scope

This document describes supervisor customer-management in `cpanel/`.

It covers:
- the customers list route,
- the customer detail/update route,
- route-state persistence for list filters/sort/page,
- the shared control-panel form primitives extracted during the feature,
- the related shell/navigation adjustments required to keep the feature deterministic,
- and the path-level contract inventory for customer-management surfaces.

## 2) Routes

Customer-management routes:

| Identify | Path | Role |
|----------|------|------|
| `Customers` | `/customers` | Supervisor customer list page with `customerStats.total_count` stat and a route-state-backed table. |
| `Customer` | `/customer/:formType(show)/:id` | Supervisor customer detail/update form page backed by requester reads/writes. |

Form-route convention:
- the single-record page uses the `formType` path segment style,
- `CustomerFormType` is `"show"` only,
- the page returns through `useNav().back()` instead of reconstructing list query state inside the detail page.

## 3) Customers list page

The list route is owned by `Customers.tsx`, but the durable feature logic lives in `CustomersTable.tsx` and `CustomerStatsSection.tsx`.

### 3.1 Top summary section

The page renders a stats section above the table using `customerStats.total_count` from supervisor GQL.

Card:

- total customers (`customerStats.total_count`)

The card is adapter-backed through the cpanel GQL data-adapter path.

### 3.2 Table behavior

The customer table supports:
- paginated list reads,
- free-text search across all customer fields exposed by the backend filter,
- scope-specific narrowing for email-only or mobile-only search,
- deterministic sort selection,
- route query persistence for search/scope/sort/page through `useWithHistoryState({key: "customers"})`,
- and row navigation into the single-customer route.

The row action remains visually at the end of the table, while the column label stays `الإجراءات` and the action text stays `عرض`.

### 3.3 Table UI ownership

The customer list uses the shared `DataTable` product surface.

The list uses `DataTable` for:
- search chrome,
- toolbar filter groups,
- pagination controls,
- row-count and page metadata,
- and loading/empty/data state composition.

`CustomerIdentityCell` is the small customer-specific cell renderer extracted for repeated identity presentation only; the rest of the list orchestration stays in `CustomersTable.tsx`.

## 4) Customer detail/update page

The single-customer route is owned by `Customer.tsx`.

Behavior:
- reads route params through `useCurrentParams`,
- creates a route-scoped `useShallowForm` against `API.FORMS.SUPERVISOR.R("customer")`,
- issues `customer.read` on enter for `formType = "show"`,
- maps UI-facing `isLoading` / `saving` flags inside the form `mapState(...)` instead of leaking raw form sub-state into the page body,
- renders the shared subpage header,
- reuses the shared field/button primitives,
- submits through `customer.update`,
- and returns through `back()` after successful save.

Important Pattern:
- the page keeps orchestration local,
- it does not recreate list query state,
- it does not wrap the page in a dedicated feature-only form component,
- it uses shared form submit primitives instead of HTML-form wrapper semantics,
- and its first-load state keeps the full card height reserved so the loading surface does not collapse into a short top strip.

## 5) Shared form primitives introduced by this feature

Shared form primitives:

- `FormActionButton.tsx`
- `FormTextField.tsx`

Usage:
- `Login.tsx` uses `FormActionButton`,
- `AuthTextField.tsx` delegates to `FormTextField`,
- `Customer.tsx` uses the same shared primitives.

The intent is to share the control-panel form language across future requester-backed pages without creating one wrapper component per feature.

## 6) Shell and navigation adjustments coupled to the feature

Shell and navigation contract for customer routes:

- `MainLayout.tsx` renders one shared shell tree; desktop vs mobile behavior is controlled through CSS/media queries, not separate layout branches.
- `layouts/main-layout/drawer.ts` maps both `Customers` and `Customer` to the same active drawer item so the customer area stays highlighted on the form route too.
- `Header.tsx` current subpage behavior remains the dedicated back-oriented variant used by `Customer.tsx`.
- `routes.ts` registers customer list/detail route ownership per `supervisor-admin-modules.md`.

## 7) Generated and mirrored artifacts

Mirrored supervisor artifacts the customer screens consume:

- mirrored supervisor SDL under `src/types/gql/definitions/supervisor.graphql`,
- mirrored/generated supervisor GraphQL types under `src/types/gql/gql-types/supervisor.ts`,
- incremental TS build artifact under `lib/tsconfig.tsbuildinfo`.

These files are verification artifacts and must stay aligned with backend truth and local `yarn type-check` output.

## 8) Contract inventory

| Path | Contract role |
|------|---------------|
| `cpanel/src/resources/configs/routes.ts` | Registers `Customers` and `Customer` routes with `:formType(show)/:id` pattern. |
| `cpanel/src/types/customer.ts` | Shared customer route/list query contracts (`CustomerFormType`, `CustomersRouteQuery`, `CustomersFilterScope`). |
| `cpanel/src/types/gql/definitions/supervisor.graphql` | Local mirror of the backend supervisor GraphQL SDL expanded for customer list/detail/stats reads. |
| `cpanel/src/types/gql/gql-types/supervisor.ts` | Local generated GraphQL type mirror for the supervisor schema. |
| `cpanel/src/app/ui/pages/Customers.tsx` | Route-owned customers page that composes the page title, stats section, and list table. |
| `cpanel/src/app/ui/pages/Customer.tsx` | Route-owned single-customer form page that orchestrates `customer.read` / `customer.update` through `useShallowForm` and returns through `back()`. |
| `cpanel/src/app/ui/components/customer/customers.graphql` | GraphQL operation file for the supervisor customer list read. |
| `cpanel/src/app/ui/components/customer/customer.graphql` | GraphQL operation file for the supervisor single-customer read. |
| `cpanel/src/app/ui/components/customer/CustomersTable.tsx` | Customer-list orchestration owner for route-state query binding, adapter query construction, table columns, and row navigation. |
| `cpanel/src/app/ui/components/customer/CustomerStatsSection.tsx` | List header stat from `customerStats.total_count`. |
| `cpanel/src/app/ui/components/customer/CustomerIdentityCell.tsx` | Small reusable identity cell renderer for avatar/initials plus name/email presentation inside the table. |
| `cpanel/src/app/ui/components/DataTable.tsx` | Shared table surface for customer-list search/filter/pagination UX. |
| `cpanel/src/app/ui/components/form/FormActionButton.tsx` | Shared form action button with hover/cursor/loading behavior used by login and customer pages. |
| `cpanel/src/app/ui/components/form/FormTextField.tsx` | Shared form text input primitive built on `useFormInput` and `FormInputWrapper`. |
| `cpanel/src/app/ui/components/auth/AuthTextField.tsx` | Auth field delegates to `FormTextField`. |
| `cpanel/src/app/ui/components/auth/AuthSubmitButton.tsx` | Auth submit button; login uses `FormActionButton`. |
| `cpanel/src/app/ui/pages/Login.tsx` | Login consumes `FormActionButton` for aligned submit behavior. |
| `cpanel/src/app/ui/components/Header.tsx` | Shared header current subpage behavior supports the customer form page without duplicating theme-switch controls. |
| `cpanel/src/app/ui/layouts/MainLayout.tsx` | Responsive shell renders one branch at a time, preventing duplicated route work on first load. |
| `cpanel/src/app/ui/layouts/main-layout/drawer.ts` | Drawer-route ownership mapping keeps the customer area active for both list and detail routes. |
| `cpanel/src/resources/translations/ar.ts` | Arabic copy expanded for customer pages, customer stats, and the customer table labels/options. |
| `cpanel/lib/tsconfig.tsbuildinfo` | Incremental TypeScript verification artifact from cpanel type-check. |

## 9) Documentation and Governance Artifacts

| Path | Contract role |
|------|---------------|
| `docs/platforms/cpanel/README.md` | Cpanel documentation index with customer-management entry | `docs/platforms/cpanel/README.md` |
| `docs/platforms/cpanel/overview.md` | Platform overview with customer module routes and backend coupling | `docs/platforms/cpanel/overview.md` |
| `docs/platforms/cpanel/data-flow-and-gql.md` | Adapter-backed GQL read flows for the customer module | `docs/platforms/cpanel/data-flow-and-gql.md` |
| `docs/platforms/cpanel/repository-inventory.md` | Repository inventory with customer-related paths | `docs/platforms/cpanel/repository-inventory.md` |
| `docs/platforms/cpanel/customer-management.md` | Customer module behavior, path ownership, and contract inventory | `docs/platforms/cpanel/customer-management.md` |
| `docs/platforms/backend/contracts/supervisor-customers-and-stats.md` | Backend supervisor customer read surface and bridge ownership | `docs/platforms/backend/contracts/supervisor-customers-and-stats.md` |
| `.cursor/rules/cpanel-platform-governance.mdc` | Durable cpanel governance with form-page constraints | `.cursor/rules/cpanel-platform-governance.mdc` |

## Related

- `docs/platforms/cpanel/overview.md`
- `docs/platforms/cpanel/data-flow-and-gql.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
