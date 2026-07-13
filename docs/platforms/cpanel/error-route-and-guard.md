# CPanel Error Route and Guard Behavior

Target supervisor cpanel contract (see `overview.md`). Describes the error route and guard bypass for the supervisor cpanel contract.

## Purpose

This page documents the error-route surface in `cpanel/` and the paired router-middleware adjustment that allows the route to render as an explicit fallback page.

## Outcomes

Contract outcomes when the checkout implements the error route:

- `Error` renders a branded centered card on `BASIC` using `Logo`, semantic color/dimension tokens, and `Utils`.
- Route accepts `403`, `404`, and `500` through `/:error(404|500|403)`.
- Visual hierarchy: logo, large numeric error code, translated description, primary action button to `Home`.
- Arabic translations include dedicated `redirectHome` action label.
- Router middleware exits immediately when current page identify is `Error`, preserving the fallback page.

## Route and Runtime Contract

### Route shape

`cpanel/src/resources/configs/routes.ts` still declares the route as:

- `Error` -> `/:error(404|500|403)` -> `BASIC`

This is a narrow platform fallback route for explicit error codes, not a generic status-page system for every HTTP code.

### Guard bypass behavior

`cpanel/src/app/services/router.ts` checks `currentIsErrorPage(myInstance)` at the top of `applyRouterMiddleware(...)` and returns early.

Effect:

- the page can render even when the normal auth/public-route checks would otherwise redirect,
- the route is treated as a terminal fallback surface instead of another page to gate,
- the existing `Login`/`Home` redirect rules continue to apply to non-error pages only.

This change is operationally important because the error page is meant to communicate the failure state already resolved by routing/runtime, not trigger another redirect cycle.

## Error Page UI Surface

`cpanel/src/app/ui/pages/Error.tsx` is a route-owned branded error surface built on the cpanel UI foundation.

Behavior:

- reads the route param through `useCurrentParams(...)`,
- binds `useTranslator(...)` to `t.ui.pages.error`,
- reads the color scheme through `useThemeManager()`,
- derives solid semantic action fills (`semanticColor.primaryActionBackground`, `semanticColor.accentActionBackground`) from `theme.ts`,
- shows the actual error code as the page title,
- renders the translated error description below the code,
- redirects to `Home` through `useRouter().redirect(...)` when the primary action is clicked.

### Visual composition

The route keeps all visual composition local to the page and reuses existing shared primitives rather than introducing a new shared component.

Page-owned composition includes:

- a centered `Container`,
- a card shell using semantic backgrounds, borders, shadows, and spacing,
- two blurred decorative solid-color circles,
- a pill-shaped logo holder,
- a large display treatment for the numeric code,
- a single primary action button using the existing solid/reset helpers.

This is still route-owned UI, not a shared multi-screen error component.

## Translation Contract

`cpanel/src/resources/translations/ar.ts` remains the source of truth for error-page copy.

Route-owned error keys include:

- `403`
- `404`
- `500`
- `redirectHome`

Other status-code descriptions remain present in the translation file, but the `Error` route does not expose them because the route pattern still limits supported params to `403|404|500`.

## Generated Artifact

`cpanel/lib/tsconfig.tsbuildinfo` is a generated incremental TypeScript artifact from `tsc --noEmit` verification.

It is accounted for here explicitly but not narrated line-by-line because it is a generated incremental TypeScript artifact rather than a source-of-truth implementation file.

## Contract inventory

The table below inventories every tracked path for the error-route contract.

| Path | Contract role | Where documented |
|------|-------------------------------------|------------------|
| `cpanel/src/app/services/router.ts` | Early return for `Error` inside router middleware so fallback pages are not redirected again | `# Route and Runtime Contract` / `### Guard bypass behavior` |
| `cpanel/src/app/ui/pages/Error.tsx` | Route-owned branded error page with logo, numeric code, translated message, and redirect action | `# Error Page UI Surface` |
| `cpanel/src/resources/translations/ar.ts` | Centralized Arabic error-route action label plus status descriptions | `# Translation Contract` |
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated incremental TypeScript artifact touched by verification | `# Generated Artifact` |
| `docs/platforms/cpanel/error-route-and-guard.md` | Error-route contract page | this page |
| `docs/platforms/cpanel/README.md` | Cpanel documentation index | `docs/platforms/cpanel/README.md` |
| `docs/platforms/cpanel/overview.md` | Platform overview with error route and router guard contract | `docs/platforms/cpanel/overview.md` |
| `docs/platforms/cpanel/repository-inventory.md` | Repository inventory with `Error` page paths | `docs/platforms/cpanel/repository-inventory.md` |
| `docs/platforms/cpanel/component-structure.md` | Page ownership guidance for the `Error` route | `docs/platforms/cpanel/component-structure.md` |
| `docs/platforms/cpanel/ui-foundation.md` | UI foundation reference for the `Error` route | `docs/platforms/cpanel/ui-foundation.md` |
| `docs/platforms/cpanel/data-flow-and-gql.md` | Screen responsibility boundaries for the error route | `docs/platforms/cpanel/data-flow-and-gql.md` |
| `docs/invariants/cpanel.md` | Cpanel invariants including error-route placeholder discipline | `docs/invariants/cpanel.md` |
| `.cursor/rules/cpanel-platform-governance.mdc` | Durable cpanel governance with prop-first `Utils` usage | `.cursor/rules/cpanel-platform-governance.mdc` |
| `.cursor/skills/cpanel-platform-governance/SKILL.md` | Repeatable cpanel workflow with prop-first `Utils` guidance | `.cursor/skills/cpanel-platform-governance/SKILL.md` |
