# Ejtmaa Website Invariants

## Purpose

Non-negotiable implementation invariants for the Customer SSR web frontend under `website/`.
Complements backend and cpanel invariants.

Actors: **visitor** and **customer**.

Interpretation rule:
- invariants below define obligations for the customer portal contract,
- they must not be misread as proof that every route or UI surface is already implemented in the checked-in `website/` scaffold today.

## W1. Repository Ownership

`website/` is its own git repository. Git operations for website files run from `website/`.

## W2. SSR Bootstrap Authority

Runtime entry centered on:
1. `src/client/*`
2. `src/server/*`
3. `src/resources/configs/web-core.ts`
4. `src/app/ui/base/core/MyApp.tsx`
5. `src/app/ui/base/core/MyHtml.tsx`

## W3. Folder Ownership

| Path | Owns |
|---|---|
| `src/resources/` | Config, translations, shell resources |
| `src/app/services/` | Behavior orchestration |
| `src/app/ui/base/` | Framework infrastructure |
| `src/app/ui/components/` | Shared product UI |
| `src/app/ui/layouts/` | Shell layouts |
| `src/app/ui/pages/` | Route pages |
| `src/types/` | Type contracts |

## W4. UI Foundation

Mandatory foundation:
1. `ui/base/components/Utils.tsx`
2. `resources/configs/theme.ts` — navy `#0B2057`, orange `#EC6901`
3. `resources/configs/utils.ts`
4. Shared components under `ui/components/`

Use `SemanticGradients.primary` / `SemanticGradients.accent` from `theme.ts`.

## W5. Base vs Product UI

`ui/base/` is infrastructure only. Business UI belongs in `ui/components/`.

## W6. Page Orchestration

Route pages orchestrate hooks, layouts, adapters, and shared UI. Extract reusable sections to `ui/components/`.

## W7. Backend Coupling

- Mount: `/website` only
- Requesters: `auth` (visitor), `customer`, `notification` via `requesters.website.ts`
- GQL mirrors: `base` + `customer` under `src/types/gql/**`

## W8. Route Registry

Customer router only. Fixed URL segments before parametric routes on the same prefix.

## W9. Auth Flow

Customer auth on `/website`: Login, Register, ResetPassword.
Role home redirect: authed customer → customer home; visitor → public home.
Auth gradients use `SemanticGradients.primary` / `SemanticGradients.accent`.

## W10. Locale and Theme

Locale switch: cookie + full reload via `changeLocale`. No client-side locale state.
Theme paths via `ThemeMap` and `utils.ts` helpers.

## W11. Form Success Toast

`ResMainMessageMiddleware` auto-shows backend main messages. Do not duplicate in `afterSentSuccess`.

## W12. Static Info Pages

About page keys use `aboutEjtmaa`.

## W13. Documentation Discipline

Material architecture changes update website docs in the same task.

## W14. Page Title Translation

Per-page `<title>` values come from page-scoped `useTranslator` keys mirrored in both `ar.ts` and `en.ts`. No hardcoded title literals in JSX.

## W16. SSR Boot Alignment

SSR boot follows W2 entry stack. Startup hydration uses `/website/custom/start` auth payload shape documented in `ssr-boot-and-startup.md`.

## W18. Form Requester Map

Website forms resolve through `requesters.website.ts` and `FORMS.*` maps.

## W19. Shared UI Foundation

Shared shell primitives use W4–W5 foundation (`Utils`, `theme.ts`, `ui/components/`).

## W20. Shell Folder Ownership

Visitor and customer shell components follow W3 folder ownership; do not place product UI in `ui/base/`.

## W22. Role Redirect Alignment

Post-auth redirects follow `route-registry-contract.md` §6 role-home rules.

## W24. Route-Reactive Memoization

Components deriving UI from `currentPage` must not use `withMemo` with no props. See `.cursor/rules/website-route-reactive-components.mdc`.

## W25. Flex Text Clamp

`maxLi` requires a bounded width. Flex-growing labels use block ellipsis instead.

## W26. Customer Shell Layout Padding

Customer main layout respects `Dims.headerHeight` / `Dims.bottomBarHeight` for content padding.

## W27. Customer Drawer RTL

Portaled drawer slides from the start side in LTR and end side in RTL.

## W28. Card Layout Spacing

Customer list cards use `semanticDims.card` spacing tokens and the two-group content pattern.

## W29. Utils Props Over baseCssStyle

Prefer `Utils` component props; reserve `baseCssStyle` for values without Utils equivalents.

## W30. Honest Card Data

Do not render mock fallback maps as real listing data when GQL has no field.

## W35. Breadcrumb rootLabel Precedence

When a descendant route provides `rootLabel` for the current parent target, `useBreadcrumbs` prefers it and stops the walk.

## W36. One Adapter Per Route

Each route page owns one primary adapter hook; split screens share a body component.

## W37. Route Registry Contract

Customer routes register through `customerRouter` in `routes.ts` per `route-registry-contract.md`.

## W38. Drawer Subpage Breadcrumb

Authed drawer subpages without bottom bar must declare route `breadcrumb` before ship.

## W40. Deck Stacking Isolation

Deck hosts with high intra-deck `zIndex` use `isolation: isolate` so portaled overlays stay above.

## W41. Static-Before-Parametric Routes

Fixed URL segments register before `:id` routes on the same prefix. See `route-registry-contract.md` §4.

## W42. Presentational Label Props

Presentational components accept translated label strings from callers. Callers own `useTranslator` scope; presentational components do not call `useTranslator` for labels passed as props.

See `.cursor/rules/website-presentational-label-props.mdc`.

## Related

- `docs/platforms/website/overview.md`
- `.cursor/rules/website-platform-governance.mdc`
