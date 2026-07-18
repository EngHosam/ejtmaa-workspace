# Ejtmaa Website Invariants

## Purpose

Non-negotiable implementation invariants for the Customer SSR web frontend under `website/`.
Complements backend and cpanel invariants.

Actors: **visitor** and **customer**.

Interpretation rule:
- invariants below define obligations for the customer portal contract,
- they must not be misread as proof that every route or UI surface is already implemented in the checked-in `website/` scaffold today.

## Shipped baseline (current state)

The checked-in scaffold currently ships: actor model `CUSTOMER` + `visitor` on mount `/website`; `AuthedAs = "CUSTOMER"`; routes `Login`, `Register`, `ResetPassword`, `Home`, `UiMockup`, `Error` (`publicRoutes = ["Login", "Register", "ResetPassword", "UiMockup", "Home"]`, `getMyHomeIdentify = "Home"`); layouts `BASIC`, `LANDING`, `MAIN`; GQL mirror `base` + `customer`; requesters `visitor.auth` (`registerCustomer`, `login`, `resetPassword`), `customer.customer`, `customer.notification`; socket namespace `customer`; auth pages `Login`, `Register`, `ResetPassword` on split `AuthPageShell` with `API.FORMS.R("auth")`; `BASE_URL` on `/website`.

Not yet shipped: Google social auth, `CustomerHome`, `CUSTOMER_MAIN` layout, `customerRouter`, `/customer/*` workspace, `CustomerHeader`/`CustomerDrawer`/`CustomerBottomBar`, customer settings/notifications/help/about/terms screens, `CUSTOM.START` boot wiring, customer role home redirect. These remain obligated by the invariants below and the flow docs.

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

## W29. Utils Style Prop Precedence (baseCssStyle vs cssStyle)

Inside `Utils` `Box.css()`, layers merge in order — later wins: reset → `baseCssStyle` → shorthand props (`bg`,`clr`,`p`,`w`,`h`,`crn`,`br`,`shd`,`fs`,`fw`,`clp`,`ta`,`opc`,`asp`,…) → `cssStyle` → `disabledStyle`/`hidden`/`hideAt`/`showAt`.

- `baseCssStyle` is **overridden** by shorthand props. Use it ONLY for `...ElementStyles.buttonReset`, or a genuine default a shorthand must override (the conditional-fill `background:"transparent"` under `bg={active?X:undefined}`). Nothing else belongs here — not `:hover`, `@media`, `transition`, `transform`, `cursor`, `lineHeight`, `letterSpacing`, `overflowX/Y`, `boxShadow`, `display`, `flex`, `gridTemplateColumns`, `backgroundImage`, logical `textAlign`, asymmetric `borderRadius`, `backdropFilter`, `pointerEvents` (none have a conflicting shorthand, so they belong in `cssStyle`).
- `cssStyle` **overrides** shorthand props. Put every regular no-shorthand value here. To override a `clr`/`bg` forced by a wrapper, use `cssStyle={{color}}`/`{{background}}` — it wins over the shorthand.
- Prefer a shorthand prop over either layer when one exists (`fs` not `fontSize`, `opc` not `opacity`, `maxW` not `maxWidth`, `minW` not `minWidth`, `mt` not `marginTop`, `crn` not `borderRadius`, `br` not `border`, `bg` not `background`, `clr` not `color`, `asp` not `aspectRatio`).
- `cursor:"pointer"` is auto-added by Utils when the `onClick` prop is set — do not duplicate it. Set `cursor` explicitly only for `extra.onClick` wiring or non-pointer cursors.
- A reusable component MUST expose `cssStyle` (the override) to consumers, never `baseCssStyle`. A consumer-injected `baseCssStyle` would be silently overridden by the component's shorthand props — an unreliable escape hatch.

See `.cursor/rules/website-utils-style-prop-precedence.mdc` and `.cursor/skills/website-utils-style-prop-audit/SKILL.md`.

`Container`/`FlexContainer` in `Utils.tsx` resolve `maxW`/`w` through `rem()` and use `maxW != null` (not truthy) for the fallback: `maxWidth: maxW != null ? rem(maxW) : _.maxWidth`. So `maxW={0}` is honored (no falsey fallback to `_.maxWidth`), and a numeric `maxW` becomes `rem`. This is why `maxW={46}` in `SectionShell` yields `46rem`. Do not re-introduce `maxW || _.maxWidth`.

## W30. Honest Card Data

Do not render mock fallback maps as real listing data when GQL has no field.

## W35. Breadcrumb rootLabel Precedence

When a descendant route provides `rootLabel` for the current parent target, `useBreadcrumbs` prefers it and stops the walk.

## W36. One Adapter Per Route

Each route page owns one primary adapter hook; split screens share a body component.

## W37. Route Registry Contract

Customer routes register through `customerRouter` in `routes.ts` per `route-registry-contract.md`.

## W38. Drawer Subpage Breadcrumb

Authed drawer subpages without bottom bar must declare route `breadcrumb` before ship. `CustomerMainLayout` renders fixed `CustomerSubHeader` from `routes[identify]?.breadcrumb` and offsets content by `CUSTOMER_SUBHEADER_BAR`. Breadcrumb product code (`Breadcrumb`, `useBreadcrumbs`, `HomeMark`, `CustomerSubHeader`) lives under `ui/components/`, not `ui/base/`. See `flow-customer-shell.md` §7.1.

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

## W49. Native `<button>` Background Neutralization

`ElementStyles.buttonReset` resets `appearance`/`border`/`outline` but NOT `background`. A `Row`/`Col`/`Box` rendered `As={"button"}` keeps the browser default (light/white) button background, which shows through in dark mode when no `bg` shorthand is set. Every `As={"button"}` MUST neutralize it by one of three patterns:

- **Always-filled** (both states filled): `bg={...}` shorthand overrides the native bg.
- **Conditionally-filled** (`bg={active ? X : undefined}`): `background:"transparent"` in `baseCssStyle` (alongside `...buttonReset`) so the `bg` shorthand wins for active; inactive falls back to transparent. Never put this transparent in `cssStyle` — it would override the `bg` shorthand and lose the active fill.
- **Always-transparent** (outline/ghost, never filled, no conditional `bg`): `bg="@transparent"` shorthand. Do NOT use `background:"transparent"` in `baseCssStyle`/`cssStyle` here — `bg` is the shorthand for `background`, so the shorthand is the intent-revealing choice. (`baseCssStyle` background-transparent is reserved for the conditional-fill case where a `bg` shorthand must be able to win.)

See `.cursor/rules/website-utils-style-prop-precedence.mdc` § Native `<button>` background.

## W50. `rem()` Numeric Shorthand Convention

`rem()` in `Utils` converts a **number** to a `rem` string and passes strings through unchanged. Pass a **number** to any rem-based shorthand when a rem value is intended (`fs={0.8}`, not `fs={"0.8rem"}`; `minW={1.5}`, `maxW={36}`, `mt={-0.15}`). Pass a **string** only for non-rem values (`"100%"`, `"1 / 1"`, `"0.85rem 1.25rem"`, `"0 0 1rem 1rem"`). Unitless `lineHeight` multipliers (`1.3`, `1.5`) MUST live in `cssStyle` — the `lh` prop forces `rem()` and would turn `1.3` into `1.3rem`. Raw SVG attributes (`<circle height="...">`) are NOT Utils shorthands and keep their unit strings — do not run rem-number conversion through them.

See `.cursor/rules/website-utils-style-prop-precedence.mdc` § `rem()` and numeric shorthand values.

## W51. Accent Action Text Is White

`semanticColor.accentActionText` resolves to `base.text.action.default.fill.accent` = `BaseColors.white` in **both** light and dark schemas (`theme.ts`). Text/icons on a solid accent (orange `#EC6901`) fill MUST be `accentActionText` (white) — never `textPrimary`/`textSecondary` (dark ink), which is invisible on orange. This is a brand-authority decision in `theme.ts`; `accentActionText` is white on purpose, not a token to "fix" back to a dark value. Pair `accentActionBackground` (orange) with `accentActionText` (white); pair `primaryActionBackground` (navy) with `primaryActionText` (white).

See `.cursor/rules/website-text-clr-on-colored-bg.mdc` and `docs/platforms/website/brand-identity-alignment.md`.

## W52. SSR Hydration Determinism for Computed SVG Coordinates

Server-rendered and client-rendered floating-point values for SVG attributes (`cx`/`cy`/`strokeDasharray`/`r`) MUST serialize identically or React emits a hydration mismatch. `Math.cos`/`Math.sin`/`Math.PI` can produce values that differ at the 14th decimal between Node and the browser, so any computed coordinate that lands on a raw SVG attribute MUST be rounded to a fixed precision (e.g. `round2 = n => Math.round(n*100)/100`) before being passed to the attribute. This applies to `HeroDashboard.tsx` seat coordinates and vote-ring dash arrays. Do not round Utils shorthand values (those are deterministic); only raw SVG attribute values from trig/PI computations.

See `docs/platforms/website/landing-page.md` § Hydration.

## W53. Landing Page Composition

The public landing page (`website/src/app/ui/pages/Home.tsx` → `home/HomeScreen.tsx`) is the visitor home on mount `/website`. It composes 10 sections in order (`Hero`, `Problem`, `Platform`, `Lifecycle`, `Capabilities`, `Roles`, `LiveCommand`, `Trust`, `Impact`, `Faq`) inside a `landing-layout` shell (`shell.ts`, `drawer.ts`, `footer.ts`, `useActiveLandingSection.ts`) with `LandingHeader` + `LandingMobileDrawer` + `LandingFooter`. Sections share `home/SectionShell.tsx` (tone: `plain`/`brand`/`accent`/`inverted`) and `home/Reveal.tsx` (scroll reveal). Internal nav links use the `Link` component from `@my-ssr/web-core` with a typed `Href<keyof MPagesRoutes>` (not raw string paths) for client-side `nav.push` routing; unimplemented pages pass `undefined` temporarily. The landing surface is the canonical reference for W29/W49/W50/W51/W52/W54/W55 — see `docs/platforms/website/landing-page.md` for the full section map and contracts.

See `docs/platforms/website/landing-page.md`.

## W54. Cross-Page Navigation — `Link` + Typed `Href`

Internal cross-page navigation MUST use the `Link` component from `@my-ssr/web-core` (not a raw `<a>`/`As={"a"}`). `Link` renders a real `<a href>` (SEO / middle-click / new-tab work) and intercepts the click → `getNav().push(to)` for client-side transition with no full reload. `Link` accepts `to`/`href: string | "CURRENT" | To<Ident>` where `To<Ident> = Href<Ident> & {state?}` and `Href<Ident> = {identify: Ident, replace?, sub?} & ParamsQuery<Ident>` (web-core `src/types/router.ts`). Callers MUST pass a **typed object** `{identify: "Login"}` (`Href<keyof MPagesRoutes>`), not a raw string path — the object form is type-checked against `MPagesRoutes` (`routes.ts`; current members `Login`, `Home`, `UiMockup`). Adding a navigable page = TWO edits in `routes.ts`: the `routes` map (path/layout/loader) AND the `MPagesRoutes` interface; until both exist, callers pass `href: undefined` (never a fake `identify` or guessed string). Programmatic nav from a button `onClick` uses `useNav()` (`website/src/app/ui/base/hooks/useNav`, CSR-only) → `push({identify: "X"})`; do not call it during render/SSR. `external` is for external URLs only (renders `<a target="_blank">` with no interception, full reload). Combining `Link` with Utils styling uses `<Text As={Link} extra={{href, ...}}>`; `Link` props flow through `extra`, `textDecoration` in `cssStyle`, `buttonReset` valid in `baseCssStyle`.

See `.cursor/rules/website-link-href-navigation.mdc` and `.cursor/skills/website-link-href-audit/SKILL.md`.

## W55. In-Page Section Navigation + SSR-Inert Active State

Navigation between sections of the SAME page (landing sections) is **scroll-based** (`scrollIntoView` via `scrollToLandingSection` / `scrollToLandingTop` in `landing-layout/drawer.ts`), NOT route-based — do not use `Link`/`push` for same-page section jumps and do not invent a route per section. The active section is derived client-side from `useActiveLandingSection` (`landing-layout/useActiveLandingSection.ts`): it returns `null` on SSR and on the first client paint (gated by a `mounted` flag set in `useEffect`), and only resolves to a `LandingLayoutNavKey` after `IntersectionObserver` fires (`rootMargin: -45% 0px -50% 0px`). This is non-negotiable: an active section MUST NOT be guessed during SSR (it would either mark all items active or cause a hydration mismatch). Header nav items consume this hook and set `aria-current` + an accent underline only for the resolved key; on first paint no item is active.

See `docs/platforms/website/landing-page.md` § Navigation contract.

## W56. Footer Background Is Theme-Flipped (Light-in-Light / Dark-in-Dark)

`semanticColor.footerBackground` resolves to `surface.fill.container.default.footer` in `theme.ts` and is **theme-flipped**, NOT fixed:

- Light schema: `NeutralColors[100]` (`#F7F9FC`, light) — matches the light page surface.
- Dark schema: `#040811` (dark near-black) — distinct from the dark page surface `#060B15`.

Because the footer background follows the theme (light bg in light mode, dark bg in dark mode), footer text/icons MUST use the theme-flipped text tokens — `semanticColor.textPrimary` for link text and `semanticColor.textAccent` for section titles — so they resolve dark-in-light / light-in-dark and stay readable on the matching footer surface. Do NOT use a fixed-light token (e.g. `primaryActionText`/`accentActionText` white) for footer text: in light mode the footer bg is light, so white text would be invisible. This is the inverse of W51 (which fixes text white because its fill is fixed-dark/accent in both modes). The previous light value `BrandScales.navy[950]` (dark-in-both) was a bug — it forced dark `textPrimary` onto a dark fill in light mode (dark-on-dark, invisible); the fix is the theme-flipped bg, not a text change. Both `LandingFooter.tsx` and the shared `Footer.tsx` consume `footerBackground`, so the token change applies to both consistently.

See `docs/platforms/website/landing-page.md` § Footer.

## W57. Third-Party Brand Color Exemption (MicrobandCredit)

`MicrobandCredit.tsx` renders the Microband third-party brand wordmark/dot in its official brand blue `#096fb1`. This hardcoded hex is **intentional and exempt from W43** (semantic color token discipline): it is the external Microband brand identity color, not an Ejtmaa semantic surface/text color, so it MUST NOT be replaced with a `semanticColor` token or "fixed" for dark-mode contrast. Third-party brand colors are reproduced verbatim (the brand owns the value); Ejtmaa semantic tokens describe Ejtmaa's own surfaces only. Do not add a `semanticColor` entry for it (W43 YAGNI + brand-authority). If a third-party brand color ever reads poorly on a themed surface, the fix is to change the surrounding Ejtmaa surface, never the brand color.

See `docs/platforms/website/landing-page.md` § Footer.

## Related

- `docs/platforms/website/overview.md`
- `.cursor/rules/website-platform-governance.mdc`
- `.cursor/rules/website-customer-only-import-boundary.mdc`
