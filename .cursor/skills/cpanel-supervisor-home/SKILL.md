---
name: cpanel-supervisor-home
description: >-
  Ships or extends the cpanel supervisor home dashboard: HomeBridge extras-only
  payload, aliased root meetings lists, calendar-year subscription bars, brand
  chart tokens, custom tooltip, locked Arabic hero copy. Use when changing
  SupervisorHome, HomeBridge, or home charts.
---

# CPanel supervisor home

## When to Use

- Adding or changing `SupervisorHome`, `useSupervisorHome`, `SupervisorHomeBars`, or `HomeBridge`.
- Adding a new home KPI, chart, or meeting slice.
- Reviewing whether a metric belongs on home extras vs a directory `*Stats` root.

## Instructions

1. Read `docs/platforms/cpanel/supervisor-home.md` and `.cursor/rules/cpanel-supervisor-home.mdc`.
2. Keep `HomeBridge` extras-only (Me-shaped STATIC + supervisor id). Counts prefixed. Month series is an extra, not a relation.
3. Compose meeting rows with root `meetings(filter:)` aliases. Do not add Sequelize-less `static relations`.
4. One mount-private adapter inheriting `SUPERVISOR_GQL`. Cap lists in the client after load.
5. Charts: solid brand `semanticColor` fills; custom tooltip; calendar year 1–12; numeric X ticks.
6. Copy: locked hero title/lead; eyebrow الإحصاءات; no إشراف / الحصر / نبض. `ar.ts` only.
7. `Stats` tiles may use `href` and `note`. Home KPIs come from `_Home` extras, not from counting cards.
8. Refresh C17 indexes (`supervisor-home.md`, `overview.md`, `supervisor-admin-modules.md`, C18). `yarn type-check` in `backend/` and `cpanel/`.
