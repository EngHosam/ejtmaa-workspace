# Ejtmaa Website Invariants

## Purpose

Non-negotiable implementation invariants for the Customer SSR web frontend under `website/`.
Complements backend and cpanel invariants.

Actors: **visitor** and **customer**.

Interpretation rule:
- invariants below define obligations for the customer portal contract,
- they must not be misread as proof that every route or UI surface is already implemented in the checked-in `website/` scaffold today.

## Shipped baseline (current state)

The checked-in scaffold currently ships: actor model `CUSTOMER` + `visitor` on mount `/website`; `AuthedAs = "CUSTOMER"`; routes `Login`, `Home`, `UiMockup`, `Error` only (`publicRoutes = ["Login", "UiMockup", "Home"]`, `getMyHomeIdentify = "Home"`); GQL mirror `base` + `customer` (`me`, `notifications`, `me.canDeleteNotifications`); requesters `visitor.auth`, `customer.customer`, `customer.notification` (`requesters.website.ts`); socket namespace `customer` with event `onCustomerEventDate`; main drawer `dashboard` + `logout` only; auth page `Login` (email+password, `sub: "login"`); `BASE_URL` test + prod on `/website`.

Not yet shipped: `Register`, `ResetPassword`, Google social, `CustomerHome`, `CUSTOMER_MAIN` layout, `customerRouter`, `/customer/*` workspace, `CustomerHeader`/`CustomerDrawer`/`LandingDrawer`/`CustomerBottomBar`, customer settings/notifications/help/about/terms screens, `CUSTOM.START` boot wiring. These remain obligated by the invariants below and the flow docs.

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

Use `SemanticColors.primary` / `SemanticColors.accent` from `theme.ts`.

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
Auth primary surfaces use `SemanticColors.primary` / `SemanticColors.accent`.

## W10. Locale and Theme

Locale switch: cookie + full reload via `changeLocale` (from `@my-ssr/web-core`); no client-side locale state. Current locale is read via `useMyInstance().getRouter().locale`. Theme paths via `ThemeMap` and `utils.ts` helpers.

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

## W43. Semantic Color Token Discipline

Prefer `semanticColor.<key>` (`website/src/resources/configs/utils.ts`) over `@white` / `@<BaseColor>` hardcodes whenever a semantic token exists. Pair text/icon color against the **resolved surface fill** (e.g. white text belongs on a primary/dark fill, not on `secondaryActionBackground` = navy[50]). `semanticColor` path strings are type-guarded by `ThemeMapPath` (derived from `ThemeMapType`), so `yarn type-check` rejects any path that is not a real `ThemeMap` leaf. Do not add `semanticColor` tokens without a consumer (YAGNI). `theme.ts` is brand authority and is not edited to satisfy consumers.

See `.cursor/rules/website-semantic-color-token-discipline.mdc` and `docs/platforms/website/brand-identity-alignment.md`.

## W44. URL Config Brand Alignment

`website/src/resources/configs/urls.ts` production URLs use the `ejtmaa.live` / `backend.ejtmaa.live` domain and the `/website` mount (W7). `ME_URL` production is `https://ejtmaa.live`. The test-mode `BASE_URL` branch uses the same `/website` mount segment as production.

See `docs/platforms/website/brand-identity-alignment.md`.

## W45. Logo No-Frame + Preset Sizing

The brand `Logo` (`website/src/app/ui/components/Logo.tsx`) renders a bare `Image`. Consumers MUST NOT wrap it in a border/background/padding pill — render it with alignment only (`as_fs` / `as_c` / direct flex placement). Size is selected exclusively by the `preset` prop (`header` / `drawer` / `footer` / `hero`), backed by `LOGO_SIZES`; no call-site `w`/`h` overrides. `LOGO_SIZES` sets **height only** — width is never hardcoded and is derived from the image's intrinsic ~3.4:1 aspect ratio (`width: auto`); do not re-introduce a fixed `w`. Add a new preset when a new size context appears.

See `.cursor/rules/website-logo-no-frame.mdc` and `docs/platforms/website/brand-identity-alignment.md` § Logo.

## W46. Corner Radius Token Discipline

Corner radius is centralized in `Dims` (`website/src/resources/configs/theme.ts`): `corner` `8px`, `smallCorner` `6px`, `largeCorner` `12px`, `pillCorner` `999px`. Consumers read the default card corner via `semanticDims.card.radius` (points to `Dims.corner`), not by re-typing the literal. Hardcoded `crn` rem/px literals for card/button corners are forbidden. Pills/circles use `crn={999}`. Per-corner radius uses `crn_tr` / `crn_tl` / `crn_br` / `crn_bl` (not `br_tr` / `br_bl`, which do not exist). A project-wide radius change is a single `Dims` edit, not a consumer sweep. The hero/top-right toggles (`ThemeModeSwitch`, `LanguageSwitch`) and header icon buttons (`HeaderIconButton`) use `semanticDims.card.radius` — rounded-square, not pills; `crn={999}` is reserved for chips, accent bars, avatars, and blobs.

See `.cursor/rules/website-corner-radius-tokens.mdc` and `docs/platforms/website/ui-foundation.md` § Corner radius tokens.

## W47. No Gradients — Solid Semantic Colors Only

The website UI uses **only solid semantic colors**. `theme.ts` exports no gradient API (no `GradientDef` / `Gradients` / `SemanticGradients` / `getSemanticGradient` / `getGradientBackground`). `linear-gradient` / `radial-gradient` / `conic-gradient` are forbidden in every style path — including decorative overlays, clipped text, and scrollbar thumbs. Primary/accent fills use the solid tokens `semanticColor.primaryActionBackground` (navy `#0B2057`) and `semanticColor.accentActionBackground` (orange `#EC6901`), resolved via `getColor` / Utils props. `yarn type-check` guards `semanticColor` path validity; a grep for `gradient|Gradient|linear-gradient|radial-gradient|conic-gradient` under `website/src` must return no matches.

See `.cursor/rules/website-no-gradients.mdc` and `docs/design-color-system.md` § Solid colors only.

## W48. Bilingual Locale Surface (ar/en)

The website ships two locales configured in `website/src/resources/configs/web-core.ts` `localization`: `ar` (default, RTL) and `en` (LTR), with `locales: ["ar", "en"]`, `defaultLocale: "ar"`, `rtlLocales: ["ar"]`. Translation modules `resources/translations/ar.ts` (source of the `Tr` type that backs `useTranslator`) and `resources/translations/en.ts` (full mirror) MUST stay key-mirrored — every key present in `ar.ts` is present in `en.ts` with the same shape (W14 relies on this). Locale switching is cookie (`locale`) + full reload via `changeLocale(myInstance)(newLocale)` exported from `@my-ssr/web-core` only; no client-side locale state, no partial reload. The current locale is read via `useMyInstance().getRouter().locale`. The sole UI surface for switching is `LanguageSwitch` (`website/src/app/ui/components/LanguageSwitch.tsx`): a single toggle button showing the **target** language letters (`EN` when current is `ar`, `ع` when current is `en`), placed beside `ThemeModeSwitch` in the header trailing cluster, the drawer hero identity zone, and `BasicLayout`'s top-right row.

See `.cursor/rules/website-locale-switch.mdc` and `docs/platforms/website/ui-foundation.md` § Localization & RTL.

## Related

- `docs/platforms/website/overview.md`
- `.cursor/rules/website-platform-governance.mdc`
- `.cursor/rules/website-customer-only-import-boundary.mdc`
