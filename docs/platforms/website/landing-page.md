# Landing Page

Current-state reflection of the public landing page shipped under `website/`. This is the visitor home on mount `/website` (`getMyHomeIdentify = "Home"`). No change history — observable contracts and runtime facts only.

## Entry points

| Path (under `website/`) | Role |
|---|---|
| `src/app/ui/pages/Home.tsx` | Route page `Home`; renders `<HomeScreen/>` (lazy-loaded). |
| `src/app/ui/components/home/HomeScreen.tsx` | Composes the 10 sections in order; owns all `useTranslator` scopes and content arrays. |
| `src/app/ui/layouts/landing-layout/*` | Shell contracts: breakpoint media, nav keys, drawer/footer builders, active-section hook. |

`Home.tsx` is the SSR route entry; `HomeScreen` is the presentational composition root. All section strings come from page-scoped `useTranslator` keys (W14) — no hardcoded JSX labels.

## Shell — `landing-layout/`

| File | Exports / Contract |
|---|---|
| `shell.ts` | `landingShellBreakpointPx = 1100`; `landingShellDesktopMedia` (`min-width:1100px`); `landingShellMobileMedia` (`max-width:1099px`). Single breakpoint for every landing responsive split. |
| `drawer.ts` | `LandingLayoutNavKey = "platform"\|"lifecycle"\|"capabilities"\|"roles"\|"trust"\|"faq"`; `landingSectionIdMap` (DOM ids `landing-<key>`); `scrollToLandingSection` / `scrollToLandingTop` (smooth scroll); `buildLandingDrawerNavActions` / `buildLandingDrawerSections` / `buildLandingHeaderNavItems` builders. Nav is scroll-to-section, not route nav. |
| `footer.ts` | `LandingFooterLinkItem.href: Href<keyof MPagesRoutes>` (typed route object, not a string path). `footerLinkHrefs` is intentionally empty — every target page is unimplemented, so every `href` resolves to `undefined` until the page is wired in `routes.ts` + `MPagesRoutes`. |
| `useActiveLandingSection.ts` | `useActiveLandingSection()` — IntersectionObserver hook (`rootMargin: -45% 0px -50% 0px`) returning the in-view `LandingLayoutNavKey`. Returns `null` on SSR and on the first client paint (gated by a `mounted` flag set in `useEffect`), resolving only after observers fire. This SSR-inert behavior is mandatory (W55) — never guess an active section during SSR. |

## Header / Drawer / Footer

- `LandingHeader.tsx` — sticky top bar (`Fixed`, `zIndex.header`, `headerBackground`, bottom border `navigationBorder`, `minH = semanticDims.shell.headerHeight`). Mobile menu control uses shared `DrawerMenuIcon` (`src/app/ui/components/DrawerMenuIcon.tsx` — same mark as customer header; not `FiMenu`). Logo button `onClick={scrollToLandingTop}`; its `aria-label`/`title` = `mainHeader.back` (العودة) — note: a `mainHeader.scrollToTop` key exists in `ar.ts`/`en.ts` but is currently **unused** (dead) since the logo uses `back`. Desktop nav items are built from `buildLandingHeaderNavItems`; `isActive = item.navKey === activeNavKey` (from `useActiveLandingSection`, `null` on first paint → no item active); active item sets `aria-current="true"`, accent `textAccent` + bold, and an animated accent underline (`motion` scaleX). Clicking a nav item scrolls to its section (not a route). Trailing cluster: `ThemeModeSwitch` + `LanguageSwitch` + two `FormActionButton`s (`login` neutral, `register` primary) that both `push({identify: "Login"})` via `useNav()` (register page not implemented yet). `key={item.label}` on nav items. All `As={"button"}` elements neutralize the native button background per W49 (logo + nav items use `bg="@transparent"` — always-transparent pattern).
- `LandingMobileDrawer.tsx` — mobile nav drawer built from `buildLandingDrawerSections`. Drawer items are `As={"button"}` with `bg="@transparent"` (always-transparent) and a `:hover` background tint in `cssStyle`.
- `LandingFooter.tsx` — footer columns from `buildLandingFooterSections`. Internal links render as `<Text As={Link}>` (`Link` from `@my-ssr/web-core`) with a typed `Href<keyof MPagesRoutes>` passed via `extra.href` (see § Navigation contract). Link text uses `clr={semanticColor.textPrimary}` and section titles `clr={semanticColor.textAccent}` — both theme-flipped (dark-in-light / light-in-dark) because the footer bg is theme-flipped (W56). `textDecoration:"none"` lives in `cssStyle` (no shorthand); `...ElementStyles.buttonReset` is the only `baseCssStyle` content. Footer `bg = footerBackground` (theme-flipped: `NeutralColors[100]` light / `#040811` dark, W56), top border `footerBorder`. The `MicrobandCredit` **MicroBand** wordmark/dot keeps its hardcoded official brand blue `#096fb1` (third-party brand color, exempt from W43 — W57).

## Section primitives

- `SectionShell.tsx` — shared section wrapper. `tone: "plain" \| "brand" \| "accent" \| "inverted"` maps to `pageBackground` / `sectionBrandBackground` / `sectionAccentBackground` / `accentActionBackground`. Renders eyebrow (orange `textAccent` bar + label), title, lead, then `children`. `inverted` tone (orange fill) forces `accentActionText` (white) for eyebrow/title/lead (W51). `scrollMarginTop` in `cssStyle` offsets the sticky header so anchored sections aren't hidden under it. Vertical padding `4rem 0` mobile, `6rem 0` desktop (`@media` in `cssStyle`).
- `Reveal.tsx` — scroll-reveal wrapper (Framer Motion + IntersectionObserver). Used by sections for entrance animation. No style-prop escape hatch is exposed (consumer cannot inject `baseCssStyle`/`cssStyle`); it is a closed presentational primitive.

## Sections (composition order in `HomeScreen`)

| # | Component | `SectionShell` tone | Purpose / key contract |
|---|---|---|---|
| 1 | `Hero.tsx` (+ `HeroDashboard.tsx`) | — (own layout) | Brand headline, primary/secondary CTAs, trust badges, and the `HeroDashboard` SVG (live session ring + seat arrangement). CTA `onPrimaryClick` → `navigateToLogin`; `onSecondaryClick` → scroll to Platform. |
| 2 | `Problem.tsx` | `plain` | Two-card problem statement + `ChaosIllustration` SVG. Illustration structural strokes use `semanticColor.iconPrimary` (not `primaryActionBackground`) for dark-mode contrast. |
| 3 | `Platform.tsx` | `accent` | Platform pillars + `GovernanceGraph` SVG + featured card. Orange accents on `sectionAccentBackground` (pale tint) — approved brand pattern; dark titles on the pale accent tint are readable (do not "fix"). |
| 4 | `Lifecycle.tsx` | `brand` | `المسار` — desktop horizontal rail + mobile stacked stages. `NodeBadge` is `accentActionBackground` (orange) with `primaryActionText` (white) icon. Mobile stage row is `ai_c` so the badge centers vertically with its data card. |
| 5 | `Capabilities.tsx` | `plain` | Capability grid with ghost numerals (`textBrand` for dark-mode visibility). |
| 6 | `Roles.tsx` | `plain` | Role comparison; `RoleAccent` enum values `primaryAction`/`neutralAction`/`accentAction` (semantic, not raw color names). |
| 7 | `LiveCommand.tsx` | `brand` | "Live command" agenda filmstrip (numbered tiles) + console rows. Current-item strip uses `sectionBrandBackground` (not `sectionAccentBackground`) to keep dark body text readable. `fontVariantNumeric: "tabular-nums"` in `cssStyle` for numeric alignment. |
| 8 | `Trust.tsx` | `plain` | `SealHeader` on `primaryActionBackground` (navy) with `primaryActionText` (white) text via `clr` shorthand + `opc` for the dimmed statement. `Stratum` layers + asymmetric `borderRadius` in `cssStyle`. |
| 9 | `Impact.tsx` | `brand` | Stat grid + `Sparkline` SVGs. Stat numbers use `textBrand`. Raw SVG `height` keeps its unit string (W50 — raw SVG attributes are not Utils shorthands). |
| 10 | `Faq.tsx` | `plain` | Knowledge index + `FaqAnswerPanel`. `IndexItem` is the canonical conditional-fill button (`bg={active?X:undefined}` + `background:"transparent"` in `baseCssStyle`, W49). `AccentCta` orange button uses `accentActionBackground` + `accentActionText` (white, W51). `NavArrow`/`GhostCta` are always-transparent (`bg="@transparent"`). Pagination inactive dots use `shellBorder` (visible in dark mode). |

## Style & color rules applied

The landing surface is the canonical reference for the website Utils style-prop invariants. Every section was audited against:

- **W29 — baseCssStyle vs cssStyle precedence.** `baseCssStyle` holds only `...ElementStyles.buttonReset` (or the conditional-fill `background:"transparent"`). Every `:hover`, `@media`, `transition`, `transform`, `cursor`, `lineHeight`, `letterSpacing`, `fontVariantNumeric`, `textTransform`, `scrollMarginTop`, `backgroundImage`, asymmetric `borderRadius`, logical `textAlign`, `backdropFilter`, `pointerEvents` lives in `cssStyle`. No `baseCssStyle`/`cssStyle` property duplicates a shorthand.
- **W49 — native `<button>` background.** All `As={"button"}` elements neutralize the browser default: always-filled → `bg={...}`; conditional-fill → `background:"transparent"` in `baseCssStyle` + conditional `bg`; always-transparent → `bg="@transparent"` shorthand.
- **W50 — `rem()` numeric convention.** Rem-based shorthands take numbers (`fs={0.8}`, `minW={1.5}`, `maxW={46}`, `mt={-0.15}`); strings only for non-rem (`"100%"`, `"1 / 1"`, compound padding). Unitless `lineHeight` in `cssStyle`. Raw SVG attributes keep unit strings.
- **W51 — accent action text is white.** `accentActionText` = white (both schemas); orange fills pair with white text/icons, never dark ink.
- **W43 — semantic color token discipline.** No raw `rgba(...)`/`#hex`/`@white` hardcodes where a `semanticColor` token exists; borders use `getColor(semanticColor.shellBorder)` etc.

Audit workflow: `.cursor/skills/website-utils-style-prop-audit/SKILL.md`. Text/color audit: `.cursor/skills/website-text-clr-audit/SKILL.md`.

## Hydration

`HeroDashboard.tsx` computes SVG seat coordinates from `Math.cos`/`Math.sin`/`Math.PI` and passes them to raw `<circle cx/cy>` attributes, plus a vote-ring `strokeDasharray`. Node and the browser can serialize these floats differently at the 14th decimal → React hydration mismatch. Fix (W52): a `round2 = n => Math.round(n*100)/100` helper rounds `s.x`, `s.y`, `voteCircumference`, and `voteDash` to 2 decimals before they reach the attribute, so server and client serialize identical strings. Only raw SVG attribute values from trig/PI are rounded; Utils shorthand values are already deterministic.

The console warning `Permissions policy violation: unload is not allowed in this document` originates from `webpack-dev-server`'s `sockjs-client` (it binds the deprecated `unload` event). It is a dev-tooling warning, not application code, and does not appear in production builds.

## Navigation contract

Two distinct navigation modes — do not mix them.

### In-page section navigation (scroll-based, NOT route-based)

Header logo and nav items, and the mobile drawer items, navigate between sections of the SAME page via `scrollToLandingSection` / `scrollToLandingTop` (`landing-layout/drawer.ts`, `scrollIntoView({behavior:"smooth"})`). There is no route per section. The active item is derived from `useActiveLandingSection` (IntersectionObserver), which is `null` on SSR and first paint (W55) — never guess an active section during SSR.

### Cross-page navigation (`Link` + typed `Href`)

Cross-page navigation uses the `Link` component from `@my-ssr/web-core` with a **typed object**, not a raw string path (W54):

- `Href<Ident> = { identify: Ident, replace?, sub? } & ParamsQuery<Ident>`; `To<Ident> = Href<Ident> & { state? }` (web-core `src/types/router.ts`).
- `Link` accepts `to`/`href: string | "CURRENT" | To<Ident>`. Pass `{ identify: "Login" }` (`Href<keyof MPagesRoutes>`). `Link` renders a real `<a href>` (via `getHref`, so SEO / middle-click / open-in-new-tab work) and intercepts the click → `getNav().push(to)` (client-side, no full reload).
- `external` → `<a href target="_blank">` with no interception (full reload, new tab) — external URLs only.
- Programmatic nav from a button `onClick` uses `useNav()` (`website/src/app/ui/base/hooks/useNav`, CSR-only) → `push({ identify: "X" })`. Used by the header `login`/`register` buttons (`push({identify: "Login"})`) and the Hero/Faq primary CTAs (`navigateToLogin`). Do not call `useNav()` during render/SSR.
- Combining `Link` with Utils styling: `<Text As={Link} extra={{ href, title, "aria-label" }}>` — `Link` props flow through `extra`; `textDecoration:"none"` in `cssStyle`; `...ElementStyles.buttonReset` in `baseCssStyle` is valid for an anchor.

### Route registry — `MPagesRoutes`

`website/src/resources/configs/routes.ts` declares both the `routes` map (`path`/`layout`/`preLoadedPage`/`pageLoader`) and the `MPagesRoutes` interface. Current members: `Login`, `Home`, `UiMockup` (`Error` comes from web-core's `PagesRoutes`). **Adding a navigable page = TWO edits** in `routes.ts`: the `routes` map entry AND the `MPagesRoutes` interface member. Until both exist, callers pass `href: undefined` — this is exactly why `footerLinkHrefs` (`landing-layout/footer.ts`) is empty: none of the footer targets (pricing, about, contactUs, help, termsConditions, privacyPolicy, usagePolicy) are implemented yet, so every footer `href` resolves to `undefined` and the `Link` renders without a destination.

See `.cursor/rules/website-link-href-navigation.mdc` and `.cursor/skills/website-link-href-audit/SKILL.md`.

## Translation contract

`HomeScreen` declares one `useTranslator` scope per section (`hero`, `problem`, `platform`, `lifecycle`, `capabilities`, `roles`, `liveCommand`, `trust`, `impact`, `faq`) plus the page `title` scope. Every key is mirrored in `resources/translations/ar.ts` (source of the `Tr` type) and `resources/translations/en.ts` (full mirror) per W14/W48. The page `<title>` comes from the `title` key (W14). The `landingLayout` scope holds `drawerTitle`, `drawerSubtitle`, `footer.sections.*`, `footer.links.*`, `nav.*` (section labels), `actions.login`/`actions.register`. The `mainHeader` scope holds `menu`, `back` (used by the header logo + menu button), and `scrollToTop` — note `scrollToTop` is currently **unused** (the logo uses `back` despite scrolling to top; flagged as a minor naming/dead-key inconsistency). Removed dead key: `ui.layouts.landingLayout.actions.consoleTitle`.

Shared footer credit copy lives under `ui.components.microbandCredit.*`. The `brand` value MUST stay **`MicroBand`** (capital `B`) in both locale files; it is rendered by `MicrobandCredit.tsx` in landing, main, and customer footers.

## Change-set traceability

Current-state reflection of the landing-page change set. No change history. Generated build artifacts (`lib/tsconfig.tsbuildinfo`) and binary brand assets are listed but not narrated line-by-line.

### Committed (first commit → HEAD, `website/` repo)

| Path (under `website/`) | Status | Documented where |
|---|---|---|
| `src/app/ui/pages/Home.tsx` | M | § Entry points |
| `src/app/ui/components/home/HomeScreen.tsx` | A | § Entry points; § Sections |
| `src/app/ui/components/home/Hero.tsx` | A | § Sections #1 |
| `src/app/ui/components/home/HeroDashboard.tsx` | A | § Sections #1; § Hydration |
| `src/app/ui/components/home/Problem.tsx` | A | § Sections #2 |
| `src/app/ui/components/home/Platform.tsx` | A | § Sections #3 |
| `src/app/ui/components/home/Lifecycle.tsx` | A | § Sections #4 |
| `src/app/ui/components/home/Capabilities.tsx` | A | § Sections #5 |
| `src/app/ui/components/home/Roles.tsx` | A | § Sections #6 |
| `src/app/ui/components/home/LiveCommand.tsx` | A | § Sections #7 |
| `src/app/ui/components/home/Trust.tsx` | A | § Sections #8 |
| `src/app/ui/components/home/Impact.tsx` | A | § Sections #9 |
| `src/app/ui/components/home/Faq.tsx` | A | § Sections #10 |
| `src/app/ui/components/home/SectionShell.tsx` | A | § Section primitives |
| `src/app/ui/components/home/Reveal.tsx` | A | § Section primitives |
| `src/app/ui/components/LandingHeader.tsx` | A/M | § Header / Drawer / Footer — menu mark = shared `DrawerMenuIcon` |
| `src/app/ui/components/DrawerMenuIcon.tsx` | A | Shared brand mark; § Header; also `flow-customer-shell.md` §3.1 |
| `src/app/ui/components/LandingMobileDrawer.tsx` | A | § Header / Drawer / Footer |
| `src/app/ui/components/LandingFooter.tsx` | A | § Header / Drawer / Footer |
| `src/app/ui/layouts/landing-layout/shell.ts` | A | § Shell |
| `src/app/ui/layouts/landing-layout/drawer.ts` | A | § Shell |
| `src/app/ui/layouts/landing-layout/footer.ts` | A | § Shell; § Navigation contract |
| `src/app/ui/layouts/landing-layout/useActiveLandingSection.ts` | A | § Shell |
| `src/resources/configs/routes.ts` | M | route-registry-contract.md |
| `src/resources/configs/theme.ts` | M | W51 (`accentActionText` = white); W56 (`container.default.footer` light `BrandScales.navy[950]`→`NeutralColors[100]`, theme-flipped footer bg); brand-identity-alignment.md |
| `src/app/ui/base/components/Utils.tsx` | M | `Container`/`FlexContainer` `maxW`/`w` now use `rem()` + `maxW != null` fallback (W50); no more `maxW \|\| _.maxWidth` falsey bug |
| `src/resources/styles/dependencies.scss` | A | (Swiper import retained) |
| `src/resources/translations/ar.ts` | M | § Translation contract |
| `src/resources/translations/en.ts` | A | § Translation contract; W48 |

### Uncommitted audit pass (working tree, `website/` repo)

Style-prop audit applied the W29/W49/W50 invariants across shared components and a subset of landing files. No visual change intended.

| Path (under `website/`) | Change summary | Documented where |
|---|---|---|
| `src/app/ui/components/home/HeroDashboard.tsx` | `round2` hydration fix on SVG coords | § Hydration; W52 |
| `src/app/ui/components/home/Faq.tsx` | `rem` strings→numbers; conditional-fill button pattern; `bg="@transparent"` for always-transparent buttons; inactive dots `shellBorder` | § Sections #10; W49/W50 |
| `src/app/ui/components/home/Lifecycle.tsx` | `rem` strings→numbers; `NodeBadge` icon `primaryActionText`; mobile `ai_c` centering | § Sections #4 |
| `src/app/ui/components/home/SectionShell.tsx` | `maxW` strings→numbers | § Section primitives; W50 |
| `src/app/ui/components/home/Trust.tsx` | `color`/rgba → `clr`/`opc` shorthands (W43/W51) | § Sections #8 |
| `src/app/ui/components/LandingHeader.tsx` | `baseCssStyle`→`cssStyle`/shorthands; `bg="@transparent"`; `key={navKey}`; `scrollToTop` aria; menu `FiMenu` → shared `DrawerMenuIcon` | § Header; W29/W49 |
| `src/app/ui/components/LandingMobileDrawer.tsx` | `rem` strings→numbers; `bg="@transparent"`; `:hover` in `cssStyle` | § Drawer; W49/W50 |
| `src/app/ui/components/DataTable.tsx` | table display/borderCollapse/tableLayout → `cssStyle`; `minW`/`h`/`ws_no`/`ovr_x`/`opc` shorthands; input `bg="@transparent"` | W29/W49/W50 |
| `src/app/ui/components/Drawer.tsx` | scrollbar pseudo + `transition`/`boxShadow`/`:hover` → `cssStyle`; `mt="auto"` shorthand | W29 |
| `src/app/ui/components/Toast.tsx` | `minW`/`maxW`/`minH` shorthands; `pointerEvents`/`flexShrink` → `cssStyle`; `rem`→numbers | W29/W50 |
| `src/app/ui/components/form/FormActionButton.tsx` | `cursor`/`transition`/`:hover` → `cssStyle`; `bg` shorthand; dead `boxShadow` removed | W29 |
| `src/app/ui/components/form/FormInputWrapper.tsx` | consumer `fieldCssStyle` moved `baseCssStyle`→`cssStyle` (escape-hatch rule) | W29 |
| `src/app/ui/components/form/FormTextField.tsx` | input `bg="@transparent"`; `w`/`ta_l` shorthands; pseudos/`font`/`caretColor` → `cssStyle` | W29/W49 |
| `src/app/ui/components/auth/AuthTextField.tsx` | removed dead `fieldCssStyle={{padding}}` (was overridden by `p` shorthand) | W29 |
| `src/app/ui/components/Header.tsx` | `size` rem string→number | W50 |
| `src/app/ui/components/LanguageSwitch.tsx` | `size` rem string→number | W50 |
| `src/app/ui/components/ThemeModeSwitch.tsx` | `size` rem strings→numbers | W50 |
| `src/app/ui/components/MicrobandCredit.tsx` | `textDecoration` → `cssStyle`; `w`/`h` rem→numbers; brand blue `#096fb1` kept hardcoded (W57 third-party brand exemption) | W29/W50/W57 |
| `src/app/ui/components/Loadable.tsx` | `pointerEvents` → `cssStyle` | W29 |
| `src/app/ui/layouts/MainLayout.tsx` | `pointerEvents`/`backdropFilter`/`WebkitTapHighlightColor` → `cssStyle` | W29 |
| `src/app/ui/pages/Error.tsx` | `fs`/`bg` shorthands; `lineHeight`/`letterSpacing` → `cssStyle` | W29/W50 |
| `lib/tsconfig.tsbuildinfo` | Generated build artifact from `yarn type-check`; not narrated. | — |

### Governance artifacts (root repo, untracked)

| Path | Kind | Scope |
|---|---|---|
| `.cursor/rules/website-utils-style-prop-precedence.mdc` | rule | W29 authority — merge order, three button-bg patterns, rem convention, escape hatch |
| `.cursor/rules/website-text-clr-on-colored-bg.mdc` | rule | W51 — Text/Icn `clr` on colored fills; no dark ink on accent |
| `.cursor/rules/website-link-href-navigation.mdc` | rule | W54 — `Link` + typed `Href` cross-page navigation; `useNav().push`; `external`; add-a-page = two edits |
| `.cursor/skills/website-utils-style-prop-audit/SKILL.md` | skill | audit workflow for W29/W49/W50 |
| `.cursor/skills/website-text-clr-audit/SKILL.md` | skill | audit workflow for W51 |
| `.cursor/skills/website-link-href-audit/SKILL.md` | skill | audit workflow for W54 — raw `<a>` detection, typed `Href`, unimplemented pages |
| `docs/invariants/website.md` | invariants | W29 rewritten; W49–W55 added |
| `docs/platforms/website/landing-page.md` | doc | this page |

## Related

- `docs/invariants/website.md` — W29, W49, W50, W51, W52, W53, W54, W55
- `.cursor/rules/website-utils-style-prop-precedence.mdc`
- `.cursor/rules/website-text-clr-on-colored-bg.mdc`
- `.cursor/rules/website-link-href-navigation.mdc`
- `.cursor/rules/website-semantic-color-token-discipline.mdc`
- `.cursor/rules/website-no-gradients.mdc` (W47)
- `.cursor/skills/website-utils-style-prop-audit/SKILL.md`
- `.cursor/skills/website-text-clr-audit/SKILL.md`
- `.cursor/skills/website-link-href-audit/SKILL.md`
- `docs/platforms/website/brand-identity-alignment.md`
- `docs/platforms/website/ui-foundation.md`
- `docs/platforms/website/route-registry-contract.md`
