# CPanel Supervisor State Lanes

Shared overlay hosts under `cpanel/src/app/ui/components/`. They are not pages. Home does **not** use them (`supervisor-home.md` has its own retry row + card skeletons).

`ResultLane` is Website DNA already in the cpanel checkout. This wave added `PageStateLane` and the `FormStateLane` alias.

## 1) Which host

| Host | File | Use |
|---|---|---|
| `ResultLane` | `ResultLane.tsx` | List directories (card grid + load-more + empty/fail) |
| `PageStateLane` | `PageStateLane.tsx` | Read-only **detail** (one record) |
| `FormStateLane` | `FormStateLane.tsx` | **Same component** as `PageStateLane` (`export {default} from "./PageStateLane"`). Import this name on requester forms only. |

Do not add a third overlay implementation. Do not use `DataTable` for these directories. Do not put identity headings on `Empty` / `LaneFailed` / `Loadable`.

## 2) `PageStateLane` (detail reads)

Props: `loading`, `showEmpty`, `emptyTitle`, `emptySubtitle`, `showFailed`, `onRetry`, `children`.

Host: `Col` `relative` `flx_1` `minH` `13.75rem`. Page padding (`pt`/`gap`/`pb`) lives on the **parent** `Col`, not on the lane (moved in this wave so overlays fill leftover height).

Overlay order:

1. `loading` → centered `Loadable` `size={"3rem"}` (not card skeletons). Children are not rendered.
2. Else if `showEmpty` or `showFailed` → children not rendered; `Empty` / `LaneFailed` are absolute fills (`Wrong.tsx`). `Empty` only when `showEmpty && !loading`. `LaneFailed` only when `showFailed && !loading`.
3. Else → `children`.

Callers must **not** add props to `Empty`, `LaneFailed`, or `Loadable`. Overlay flags live on the hook (`showEmpty` = loaded-with-errors and missing record). `loading` / `isInitialLoading` must stay false when `RELOADING` **and** a record already exists (`supervisor-shared-ui.md` §5). Screen only passes flags.

**Chrome placement (detail):** identity `SectionHeading` / avatar row lives **inside** success `children`. Overlay has no heading.

Consumers:

| Screen | Hook flags |
|---|---|
| `SupervisorCustomerScreen` | `useSupervisorCustomer` |
| `SupervisorMeetingScreen` | `useSupervisorMeeting` |
| `SupervisorSubscriptionScreen` | `useSupervisorSubscription` |

## 3) `FormStateLane` (requester forms)

Identical runtime to `PageStateLane`. The alias exists so form screens stay distinguishable from GQL detail screens.

**Chrome placement (forms):** `SectionHeading` and primary/delete `FormActionButton` sit **outside** the lane (sibling above). Overlay covers fields only. Buttons stay `disabled` while `isLoading` / `showEmpty` / `showFailed`.

Consumers:

| Screen | Form |
|---|---|
| `SupervisorPlanFormScreen` | `Forms.SUPERVISOR_PLAN` |
| `SupervisorSettingsScreen` | `Forms.SUPERVISOR_SETTINGS` |

Create plan: still wrap fields in the lane (read may be a no-op / empty values). Update: `retryRead` on `LaneFailed`.

## 4) `ResultLane` (directories)

Website-shaped grid (`3 / 2 / 1` columns). Same `minH` `13.75rem`. Initial load: `renderSkeleton` (module card skeleton) × `skeletonCount` (default 6). Load-more via `LoadMoreButton`. Empty/fail: same `Empty` / `LaneFailed` overlays. Failed only when `hasErrors && !isInitialLoading && items.length === 0` (partial lists still show cards).

Consumers: `SupervisorCustomersScreen`, `SupervisorMeetingsScreen`, `SupervisorPlansScreen`, `SupervisorSubscriptionsScreen`.

Do not count `items.length` for header `Stats` (C18).

## 5) Layout host

`SupervisorMainLayout` content is a flex `Col`. Detail/form screens: `FlexContainer dir="column" flx_1` → padded `Col flx_1` wrapping the lane. Lane `flx_1` fills leftover height so overlays cover the pane.

## 6) Traceability

| Path | Role |
|---|---|
| `cpanel/src/app/ui/components/PageStateLane.tsx` | Detail overlay host |
| `cpanel/src/app/ui/components/FormStateLane.tsx` | Alias for forms |
| `cpanel/src/app/ui/components/ResultLane.tsx` | Directory grid (pre-existing DNA) |
| `cpanel/src/app/ui/components/Wrong.tsx` | `Empty` / `LaneFailed` |
| `cpanel/src/app/ui/components/Loadable.tsx` | Spinner used by `PageStateLane` |

## Related

- `.cursor/rules/cpanel-supervisor-read-directory.mdc`
- `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`
- `docs/platforms/cpanel/customer-management.md` §5 (first detail template)
- `docs/platforms/cpanel/plan-management.md`
- `docs/platforms/cpanel/supervisor-settings.md`
