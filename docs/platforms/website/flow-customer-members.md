# Website Flow — Customer Members Directory

Authenticated customer org-member directory on `CustomerMembers` (`/customer/members`). Shell/breadcrumb contract remains in `flow-customer-shell.md` §7.1; this page owns list data, search, and ResultLane UI.

## 1) Scope

**Shipped**

- Read-only member directory for the authenticated customer's organization.
- Server-side search via GQL `members(filter: { search })` (not client-side filtering of loaded rows).
- Route-query persistence under history key `members` (`useWithHistoryState`).
- Enter-to-commit search (draft local state; commit writes route query).
- Card grid via shared `ResultLane` + load-more (default `maxLoadLength: 24`).
- Chrome-only Add / Edit controls (visible; **no** nav, **no** write forms, **no** Member requesters).

**Not shipped**

- Member create/update/delete requesters or routes.
- Distinct empty copy for “no search hits” vs “no members yet” (same empty strings today).
- Page / sort / scope route params (load-more list only).

Backend filter contract: `docs/platforms/backend/contracts/member-domain.md` §4–§5.

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| Route | `CustomerMembers` → `/customer/members` (`CUSTOMER_MAIN`, `mustAuthedAs: ["CUSTOMER"]`, breadcrumb → `CustomerHome`) |
| Page | `website/src/app/ui/pages/customer/CustomerMembers.tsx` — thin `MyPage` → `CustomerMembersScreen` |
| Screen | `website/src/app/ui/components/customer/members/CustomerMembersScreen.tsx` |
| Hook | `website/src/app/ui/components/customer/hooks/useCustomerMembers.ts` |
| Card | `website/src/app/ui/components/customer/members/CustomerMemberCard.tsx` |

Helmet title: `ui.pages.customer.members.title`.

## 3) Data adapter

| Concern | Value |
|---|---|
| Mount-private adapter id | `"customer-members"` (file-private constant; not exported on `DATA_ADAPTERS`) |
| Inherited | `DATA_ADAPTERS.CUSTOMER_GQL` → `API.DATA_ADAPTERS.CUSTOMER.GQL` |
| Listable | `"members"` |
| Default page size | `initDataAdaptersProps.default.maxLoadLength` = **24** (global default changed from 50 with this slice) |
| Reload pattern | `useEffect` → `mLoad({ reload: true, query: adapterQuery })` when `adapterQuery` / `exist` / `mLoad` change; refresh + search commit reuse the same query object |

`CUSTOMER_GQL` registration (`website/src/resources/configs/store/data-adapters.ts`):

```ts
CUSTOMER_GQL: "CUSTOMER_GQL"
// init:
[DATA_ADAPTERS.CUSTOMER_GQL]: {
    api: API.DATA_ADAPTERS.CUSTOMER.GQL
}
```

GQL operation (inline in the hook):

```graphql
query CustomerMembers($filter: _MemberFilter) {
    members(filter: $filter) {
        id
        name
        email
        mobile
        avatar_url
        total_count
    }
}
```

`buildMembersAdapterQuery` omits `search` from `filter` when trimmed empty so the wire payload stays `{}` rather than `{ search: "" }`.

## 4) History search contract

Pattern aligned with Masdaria cpanel `CustomersTable` history key usage, with **Enter-only** commit (website product choice).

1. `useWithHistoryState<MembersRouteQuery>({ key: "members" })` — route nest `query.members.search`.
2. `draftSearch` local state mirrors route on `routeQuery.search` change.
3. `SearchField` `onValueChange` updates draft only; `onSubmit` (Enter) → `setRouteQuery({ search: draftSearch })`.
4. Normalized trim → `adapterQuery` → adapter reload.
5. `loadMore` / `refresh` must pass the **same** `adapterQuery` (including `filter`) so pagination does not drop the active search.

**Forbidden:** filtering already-loaded `members` in the UI; inventing write routes; omitting `filter` on load-more when search is active.

Governance: `.cursor/rules/website-customer-list-history-search.mdc`.

## 5) Screen composition

Order inside `Container` → `Col pt={2} gap={1.5} pb={2}`:

1. Row: `SectionHeading` (title + subtitle) + `FormActionButton` Add (`type="button"`, **no** `onClick` / form — chrome only).
2. `SearchField` bound to hook draft + `submitSearch`.
3. `ResultLane` with `CustomerMemberCard` via `renderCard`.

`CustomerMemberCard`: `IdentityAvatar` + name / email / mobile + Edit text button (chrome only; no nav).

## 6) Shared ResultLane chrome

Under `website/src/app/ui/components/`:

| Component | Role |
|---|---|
| `ResultLane` | Grid (3/2/1 cols), skeleton / cards / load-more / empty / failed overlays |
| `CardSkeleton` | Loading placeholder; **currently member-card shaped** (avatar + three lines + edit bone); bones use `semanticColor.inputMutedBackground` |
| `LoadMoreButton` | Centered muted button; disabled while loading more |
| `SearchField` | Card-surface input; Enter → `onSubmit` |
| `SectionHeading` | Eyebrow / title / subtitle block (also previewed on `UiMockup`) |
| `Wrong` (`Empty`, `LaneFailed`) | Absolute overlays; i18n under `ui.components.wrong.*` |

`ResultLane` requires `T extends { id: string }`. Empty/failed copy for members is passed from page i18n; generic Wrong fallbacks exist for other consumers.

**Skeleton constraint:** while `ResultLane` hard-wires `CardSkeleton`, that skeleton must match the consuming card shape. Do not reuse an unrelated listing skeleton. See `.cursor/rules/website-result-lane-skeleton-shape.mdc`.

## 7) Theme / identity

Uses existing customer semantic tokens only (`cardBackground`, `shellBorder`, `textPrimary` / `Secondary` / `Accent`, `inputMutedBackground`, `iconSecondary`, `primaryAction*`, `stateError`, `semanticDims.card.radius`). No new theme keys. `IdentityAvatar` matches shell header identity language.

## 8) Forms / write chrome

- No `useShallowForm`, no Member requester, no create/edit routes.
- Add uses shared `FormActionButton` as a **visual** primary control with `type="button"` and no handler.
- Edit is a presentational text control on the card.
- Do not wire fake success/toast for Add/Edit until requesters exist.

## 9) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.members.*` | title, subtitle, addMember, edit, searchPlaceholder, emptyTitle, emptyDescription, loadMoreBtn, loadingMoreHint |
| `ui.components.wrong.empty.*` | generic empty fallback |
| `ui.components.wrong.laneFailed.*` | failed overlay + retryLabel |
| `ui.pages.uiMockup.sections.sectionHeading` / `sectionHeadingReview.*` | UiMockup preview of `SectionHeading` |

ar/en mirrors required.

## 10) Failure / empty modes (UI)

| Condition | UI |
|---|---|
| Initial load | `CardSkeleton` grid (6) |
| Loaded empty | `Empty` overlay with members empty copy |
| Fail with no rows | `LaneFailed` + `retry` → hook `refresh` |
| More pages | `LoadMoreButton` when `thereMoreRecords` |

## 11) Traceability map (this change set)

### Backend (`backend/` repo)

| Path | Status | Doc |
|---|---|---|
| `src/app/gql/definitions/customer.graphql` | modified — `_MemberFilter`, `members(filter:)` | `member-domain.md` §4; this §1 |
| `src/app/gql/schemas/CustomerSchema.ts` | modified — pass `filter` parent | `member-domain.md` §4–§5 |
| `src/app/gql/bridges/customer/MemberBridge.ts` | modified — search `getOrmFindOptions` | `member-domain.md` §4–§5 |
| `src/app/gql/gql-types/customer.ts` | generated | narrate as codegen only |

### Website (`website/` repo)

| Path | Status | Doc |
|---|---|---|
| `src/types/gql/definitions/customer.graphql` | mirror copy | §3; `graphql-mirror-and-tooling.md` |
| `src/types/gql/gql-types/customer.ts` | mirror copy | codegen / mirror only |
| `src/resources/configs/store/data-adapters.ts` | `CUSTOMER_GQL` + default `maxLoadLength: 24` | §3 |
| `src/app/ui/pages/customer/CustomerMembers.tsx` | thin page → screen | §2 |
| `src/app/ui/components/customer/hooks/useCustomerMembers.ts` | history + adapter + Enter search | §3–§4 |
| `src/app/ui/components/customer/members/CustomerMembersScreen.tsx` | composition | §5 |
| `src/app/ui/components/customer/members/CustomerMemberCard.tsx` | card | §5 |
| `src/app/ui/components/ResultLane.tsx` | shared lane | §6 |
| `src/app/ui/components/CardSkeleton.tsx` | member-shaped skeleton | §6 |
| `src/app/ui/components/LoadMoreButton.tsx` | load more | §6 |
| `src/app/ui/components/SearchField.tsx` | Enter submit | §4, §6 |
| `src/app/ui/components/SectionHeading.tsx` | heading | §5–§6 |
| `src/app/ui/components/Wrong.tsx` | Empty / LaneFailed | §6, §10 |
| `src/resources/translations/ar.ts` / `en.ts` | members + wrong + mockup | §9 |
| `src/app/ui/pages/UiMockup.tsx` | SectionHeading preview slot | §6, §9 |
| `lib/tsconfig.tsbuildinfo` | generated | skip narrative |

### Root docs / governance (this go-doc)

| Path | Role |
|---|---|
| `docs/platforms/website/flow-customer-members.md` | this flow |
| `docs/platforms/backend/contracts/member-domain.md` | filter contract refresh |
| `docs/platforms/backend/contracts/graphql-and-types.md` | root `members(filter:)` note |
| indexes under `docs/platforms/website/*` | cross-links |
| `.cursor/rules/website-customer-list-history-search.mdc` | history + Enter + server filter |
| `.cursor/rules/website-result-lane-skeleton-shape.mdc` | ResultLane skeleton shape |
| `.cursor/skills/website-customer-result-lane-list/SKILL.md` | repeatable list workflow |

## 12) Related

- `docs/platforms/website/flow-customer-shell.md` (shell + breadcrumb)
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `.cursor/rules/website-one-adapter-per-route.mdc` (W36)
- `.cursor/rules/website-list-adapter-enter-mode.mdc`
- `.cursor/rules/website-presentational-label-props.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
