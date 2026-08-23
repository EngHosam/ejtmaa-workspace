# CPanel Customer Management

Read-only supervisor **customers** directory on `cpanel/` (backend mount `/cpanel`). Each card and detail shows that customer's **at most one** organization (`Organization.customer_id` unique). No DataTable. No home KPI widgets. No organizations module. No customer writes.

Arabic UI copy uses **مؤسسة** for that nested organization (Website DNA). Do not substitute منظمة.

Authority for similar future supervisor list/detail modules: `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md` and `.cursor/rules/cpanel-supervisor-read-directory.mdc`.

## 1) Routes and shell

`cpanel/src/resources/configs/routes.ts` — `supervisorRouter`, `SUPERVISOR_MAIN`, `mustAuthedAs: ["SUPERVISOR"]`. List **before** `:id`.

| Identify | Path | Breadcrumb |
|---|---|---|
| `SupervisorCustomers` | `/supervisor/customers` | parent `SupervisorHome` |
| `SupervisorCustomer` | `/supervisor/customers/:id` | parent `SupervisorCustomers` |

`MPagesRoutes`:

- `SupervisorCustomers: {}`
- `SupervisorCustomer: { params: { id: string } }`

Href: `{ identify: "SupervisorCustomer", params: { id } }`. Do not type a flat `id` on the route object.

Thin pages: `cpanel/src/app/ui/pages/supervisor/SupervisorCustomers.tsx`, `SupervisorCustomer.tsx` (`MyPage` → screen).

Drawer (`SupervisorDrawer`): tile after Home, `FiUser`, `alwaysAvailable`, identify `SupervisorCustomers`. Home keeps `HomeMark` `tone="onPrimary"`. No extra org/settings tiles.

Sub-header: `routes[identify]?.breadcrumb` already mounts `SupervisorSubHeader`.

Content pane (`SupervisorMainLayout`): `Col` `minH 100vh` with `paddingTop` for header / sub-header so children can `flx_1`. Footer stays a sibling after that pane.

i18n: `cpanel/src/resources/translations/ar.ts` only (`ui.pages.supervisor.customers` / `customer`, drawer `itemCustomers`). Bind nested translator nodes. No `en.ts`.

## 2) Adapters (one mount-private slot per route)

Shared registry is unchanged: `DATA_ADAPTERS.SUPERVISOR_GQL` → `API.DATA_ADAPTERS.SUPERVISOR.GQL`. **Do not** add list/detail keys to `store/data-adapters.ts` or `Forms`.

| Route | Hook | Identify | Query body |
|---|---|---|---|
| List | `useSupervisorCustomers` | stable mount-private `"supervisor-customers"` | `{ gql, listable: "customers", filter }` |
| Detail | `useSupervisorCustomer` | omitted → `GenerateIdentify()` | `{ gql, id }` — **no** `listable` |

Both: `inheritedAdapterIdentify: DATA_ADAPTERS.SUPERVISOR_GQL`. `useShallowAdapter` default `removeOnExit: true`: leaving the page removes the slot; returning injects empty and the `mLoad({ reload: true, query })` effect runs again (list rows/stats/load-more cursor are not cached). List search string can survive in history (`useWithHistoryState` key `"customers"`).

`GQLAdapterController`: `listable` → `records`; other root fields → `section`. List reads `records` as customers and `section.customerStats`. Detail reads `section.customer`.

Filter UI: `_CustomerFilter.search` only (Enter on `SearchField`). No email/mobile/sort controls.

## 3) GraphQL

Embedded operation strings in the hooks (not separate `.graphql` files). Supervisor SDL source: `backend/src/app/gql/definitions/supervisor.graphql`. Cpanel mirrors: command copy into `cpanel/src/types/gql/**` — do not hand-edit copies.

List selects: `id name email mobile avatar_url` and `organization { name status { value label } }`, plus `customerStats { total_count created_today_count }`.

Detail adds org `description logo_url subdomain custom_domain`.

Pagination window size remains adapter `maxLoadLength` (default 24). `_Customer.total_count` is the list window contract; the list **header** does not bind it.

## 4) List UI

`SupervisorCustomersScreen`: `SectionHeading`, shared `Stats`, `SearchField`, `ResultLane`.

`Stats` (`cpanel/src/app/ui/components/Stats.tsx`): `items: { id, label, value }[]`. Grid 4 / 2 / 1. Card chrome + 3px start accent rail. Values from `customerStats` extras only when they are numbers (omit the tile until then). C18: never count loaded rows.

ResultLane: `renderCard` wraps `SupervisorCustomerCard` in `Link`; `renderSkeleton` → `SupervisorCustomerCardSkeleton` (taller org line — do not reshape shared `CardSkeleton`). Load-more via `LoadMoreButton` (no DataTable pager).

Null organization: empty copy (`orgEmpty`).

## 5) Detail UI

`SupervisorCustomerScreen`: `FlexContainer` `dir="column"` `flx_1` so the lane can fill the layout `Col`. Helmet title = customer name or page title.

`PageStateLane` always uses the same outer bounds as list content: `Col flx_1 pt={2} gap={1.5} pb={2}`. Overlay states wrap an inner `Col flx_1 relative center`:

- loading: project `Loadable` `size={"3rem"}` (Website detail DNA — not card bones)
- missing: `Empty` (unchanged component)
- other errors: `LaneFailed` + hook `refresh`

Do **not** add props to `Wrong` / `Loadable`. `Empty` / `LaneFailed` stay absolute fills of that inner relative host.

Overlay flags live on the hook (screen only consumes them):

- `showEmpty` = `status === "LOADED_WITH_ERRORS" && !customer`
- `showFailed` = remaining error family and not empty
- `isInitialLoading` = `!exist || !status || LOADING || INIT`

Success: identity row + two read-only cards (customer fields; org fields or `orgEmpty`). No Forms.

## 6) Backend extras (same task)

`_CustomerStats.created_today_count: Int!` — `CustomerStatsBridge.loadExtra` → `Customer().countOfCreatedToday()` (`TODAY_START` / UTC+3 window). `total_count` remains `Customer().count()`.

## 7) Out of scope (still)

Home KPI grid; org routes/tiles; customer writes; extra filter UI; new shared adapter/Forms identifiers; hand-edited GQL mirrors.

## 8) Verification

`cpanel/` `yarn type-check` (`tsc --noEmit`). No new scripts.

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `cpanel/src/resources/configs/routes.ts` | Identifies, paths, nested params | §1 |
| `cpanel/src/app/ui/pages/supervisor/SupervisorCustomers.tsx` | Thin list page | §1 |
| `cpanel/src/app/ui/pages/supervisor/SupervisorCustomer.tsx` | Thin detail page | §1 |
| `cpanel/src/app/ui/layouts/SupervisorMainLayout.tsx` | Content `Col` + chrome pad | §1 |
| `cpanel/src/app/ui/components/supervisor/SupervisorDrawer.tsx` | Customers tile | §1 |
| `cpanel/src/resources/translations/ar.ts` | Arabic copy | §1 |
| `cpanel/src/resources/configs/store/data-adapters.ts` | Unchanged `SUPERVISOR_GQL` | §2 |
| `cpanel/src/app/ui/components/supervisor/customers/useSupervisorCustomers.ts` | List adapter + history search | §2–§4 |
| `cpanel/src/app/ui/components/supervisor/customers/useSupervisorCustomer.ts` | Detail adapter + overlay flags | §2 §5 |
| `cpanel/src/app/ui/components/supervisor/customers/SupervisorCustomersScreen.tsx` | List composition | §4 |
| `cpanel/src/app/ui/components/supervisor/customers/SupervisorCustomerScreen.tsx` | Detail composition | §5 |
| `cpanel/src/app/ui/components/supervisor/customers/SupervisorCustomerCard.tsx` | Card + skeleton | §4 |
| `cpanel/src/app/ui/components/Stats.tsx` | Shared KPI grid | §4 |
| `cpanel/src/app/ui/components/PageStateLane.tsx` | Detail load/empty/fail host | §5 |
| `cpanel/src/types/gql/definitions/supervisor.graphql` | Command-copy SDL (generated/customer-only narrative) | §3 |
| `cpanel/src/types/gql/gql-types/supervisor.ts` | Command-copy types | §3 |
| `cpanel/lib/tsconfig.tsbuildinfo` | `tsc` artifact — not product | §8 |
| `backend/src/app/gql/definitions/supervisor.graphql` | SDL source | §3 §6 |
| `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts` | Stats extras | §6 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated types | §6 |

## Related

- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/cpanel/flow-supervisor-shell.md`
- `docs/platforms/cpanel/route-registry-contract.md`
- `docs/invariants/cpanel.md` C17 C18 C19
