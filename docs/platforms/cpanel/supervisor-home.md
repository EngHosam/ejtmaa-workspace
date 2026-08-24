# CPanel Supervisor Home

Authenticated supervisor home at `/supervisor` (`SupervisorHome`). Not an empty Helmet-only page. Occupancy `Home` at `/` remains empty.

Authority for future home work: `.cursor/rules/cpanel-supervisor-home.mdc` and `.cursor/skills/cpanel-supervisor-home/SKILL.md`.

Backend payload: `docs/platforms/backend/contracts/supervisor-catalog-and-home.md` §6.

## 1) Route and copy

| Identify | Path | Layout |
|---|---|---|
| `SupervisorHome` | `/supervisor` | `SUPERVISOR_MAIN` |

Helmet: `ui.pages.supervisor.home.title` = الرئيسية.

Hero (`SupervisorHomeOpening`):

- title: متابعة المنصة
- lead (locked): نظرة على أداء وإحصائيات المنصة.
- Decorative corner brackets (small). Height follows content (do not invent `minH` to “double” the hero).
- Do **not** use إشراف or الحصر.

Stats section eyebrow: الإحصاءات.

Chart eyebrows: الاجتماعات حسب الحالة؛ الاشتراكات الجديدة حسب الشهر.

Meeting slices: بانتظار البدء then الجلسات الجارية.

Retry: تعذر التحميل / إعادة المحاولة.

`ar.ts` only. Nested translator `tr.ui.pages.supervisor.home`.

## 2) Adapter

Mount-private identify `"supervisor-home"`, inherit `DATA_ADAPTERS.SUPERVISOR_GQL`. **No** `listable`. Response lands on `section`: `home`, `startedMeetings`, `waitingMeetings` (GraphQL aliases).

One query:

- `home {` all `_Home` extras used by the screen including `new_subscriptions_by_month { year_month count }` `}`
- `startedMeetings: meetings(filter: $startedFilter)` with `status: STARTED`
- `waitingMeetings: meetings(filter: $waitingFilter)` with `status: WAITING_TO_START`

UI cap **5** per list (`HOME_MEETINGS_CAP`) after load. Do not invent a Supervisor→Meeting nested field.

Hard fail: adapter error and no `home` and no cards. Partial: retry row if errors but some payload exists.

## 3) Layout

1. Hero
2. Optional retry row
3. `Stats` tiles (optional `href`, `note`) from `home` extras — not from `customerStats` / `meetingStats` roots
4. Meeting-status `SupervisorHomeBars` (DRAFT … CANCELED)
5. New-subscription bars: calendar months **1–12** of the extra series; X-axis **numeric** ticks; tooltip uses Arabic `MMMM` from `year_month`
6. Waiting meeting cards (omit section if empty)
7. Started meeting cards (omit if empty)

Tiles:

| Tile | Extra | Href |
|---|---|---|
| العملاء | `customers_total_count` + note `customers_created_today_count` | `SupervisorCustomers` |
| الجلسات الجارية | `meetings_started_count` | `SupervisorMeetings` (unfiltered directory — honest) |
| الاشتراكات النشطة | `subscriptions_active_count` | `SupervisorSubscriptions` |
| الباقات | `plans_total_count` + note active/disabled | `SupervisorPlans` |

## 4) Charts

`SupervisorHomeBars`: Recharts `BarChart`, **solid** `semanticColor` fills (navy / orange / ink cycle). No gradients. No traffic-light `stateSuccess` / `stateError` / `stateWarning` as series colors.

Tooltip: custom content (`HomeChartTooltip`) on `semanticColor.cardBackground` / `textSecondary` / `textBrand`. Never the default Recharts tooltip (it prints English `value` and black text in dark mode).

## 5) Visual identity

Landing-like hero: `sectionBrandBackground`, 3px navy/orange corner strokes. Stats: accent rail + `textBrand` + tabular nums (`Stats.tsx`). Meeting cards reuse `SupervisorMeetingCard` + `StatusChip`.

Forbidden: Masdaria-style metric gradients, fake sparklines, the word نبض.

## 6) Failure / loading

Initial: three `SupervisorMeetingCardSkeleton`. Empty extras with no meetings still show hero after load if `home` arrived (zeros are valid).

## 7) Traceability

| Path | Role | Section |
|---|---|---|
| `cpanel/src/app/ui/pages/supervisor/SupervisorHome.tsx` | Thin page | §1 |
| `cpanel/src/app/ui/components/supervisor/home/SupervisorHomeScreen.tsx` | Composition | §3 |
| `cpanel/src/app/ui/components/supervisor/home/useSupervisorHome.ts` | Adapter query | §2 |
| `cpanel/src/app/ui/components/supervisor/home/SupervisorHomeBars.tsx` | Charts + tooltip | §4 |
| `cpanel/src/app/ui/components/Stats.tsx` | Tiles `href` / `note` | §3 |
| `cpanel/src/resources/translations/ar.ts` | Home copy | §1 |
| `backend/src/app/gql/bridges/supervisor/HomeBridge.ts` | Extras | backend catalog-and-home §6 |

## 8) Verification

`backend/` and `cpanel/` `yarn type-check`. No new scripts.

## Related

- `.cursor/skills/cpanel-supervisor-home/SKILL.md`
- `.cursor/rules/cpanel-supervisor-home.mdc`
