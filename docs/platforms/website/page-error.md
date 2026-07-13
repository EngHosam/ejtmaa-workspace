# Website Error Page

Describes the current composition and brand contract of the website error route page `website/src/app/ui/pages/Error.tsx`. Scope: `website/` only. Current-state reference (no change history).

## Route + actor

- Page class: `Error extends MyPage` (`ui/pages/Error.tsx`), rendered as the last entry in the route registry (`Error` identify).
- Actors: visitor + customer (the error page is actor-agnostic).
- Error codes supported by the page type: `"403" | "404" | "500"`. The code comes from `useCurrentParams({ ident: "Error" }).error` and defaults to `"500"` when absent.

## Translation contract

- Bound translator: `useTranslator(t => t.ui.pages.error)` in `website/src/resources/translations/ar.ts`.
- Keys consumed by the page:
  - `error.title` — page `<title>` (Helmet).
  - `error.redirectHome` — primary CTA label + `aria-label`.
  - `error["403"]`, `error["404"]`, `error["500"]` — lead description per code.
- The namespace also holds keys for other codes (`400`, `401`, `405`, `408`, `409`, `413`, `415`, `429`, `501`–`505`) not rendered by the current page type but kept for future code coverage.
- Only `ar.ts` exists under `website/src/resources/translations/` (no `en.ts` mirror today).

## Page title

The page renders `<Helmet><title>{t("title")}</title></Helmet>` inside `Main()`, satisfying `website-page-title-helmet.mdc` (W14). `MyApp` provides `defaultTitle` + `titleTemplate` (`<app title> | %s`).

## Layout composition

```
Flex (minH calc(100vh - 8rem), center, page padding)
└── Container maxW="40rem"
    └── Box (card) relative, clp, card padding, cardBackground, shellBorder,
            crn=semanticDims.card.radius, shd=4
        ├── Box (top-right corner bracket) absolute, navy, L-shape
        ├── Box (bottom-left corner bracket) absolute, orange, L-shape
        └── Col relative, as_c, maxW="28rem", ta_c
            ├── Box as_c → Logo preset="hero"
            ├── Col (code group)
            │   ├── Text (error code) — orange accent solid text
            │   ├── Box (accent underline) — orange solid bar
            │   └── Text variant="pageLead" — lead description
            └── Row (As=button) — primary CTA, navy solid fill
```

## Brand treatment

Brand pair is navy `#0B2057` (primary) + orange `#EC6901` (accent), resolved through `theme.ts` helpers — no off-brand hues. No gradients are used; all surfaces are solid semantic fills.

- **Corner brackets** — two crisp L-shaped marks replacing earlier blurred corner halos:
  - Top-right: `br_t` + `br_r` in `semanticColor.primaryActionBackground` (navy), inner corner `crn_tr="0.35rem"`.
  - Bottom-left: `br_b` + `br_l` in `semanticColor.accentActionBackground` (orange), inner corner `crn_bl="0.35rem"`.
  - Scheme-aware opacity (`opc` 0.55 dark / 0.45 light).
- **Error code** — `fw="black"`, ~5rem, solid orange text via `clr={semanticColor.accentActionBackground}`.
- **Accent underline** — short `3rem × 3px` bar, same solid orange (`getColor(semanticColor.accentActionBackground)`), `crn={999}`.
- **Primary CTA** — `Row As="button"` with `background: actionBackground` where `actionBackground = getColor(semanticColor.primaryActionBackground)` (navy), text `semanticColor.primaryActionText`, radius `semanticDims.card.radius`, `minH={semanticDims.control.height}`, `shd={3}`. On click → `redirect({ identify: "Home", replace: true })`.
- **Logo** — `Logo preset="hero"` (27 × 7.95rem), bare (no frame). See `brand-identity-alignment.md` § Logo.

## Radius + tokens used

- Card and CTA radius: `semanticDims.card.radius` (resolves to `Dims.corner`).
- Pill elements (accent underline, logo chip if any): `crn={999}`.
- Control height: `semanticDims.control.height`.
- Page/card padding: `semanticDims.page.padY`, `semanticDims.card.padding`.
- All corner radius flows through `Dims` tokens; no hardcoded `crn="1.6rem"` / `"1rem"` literals. See `website-corner-radius-tokens.mdc`.

## Verification

- `yarn type-check` (`tsc --noEmit`) in `website/` — guards `semanticColor` path validity and `Logo` preset union.
- No backend surface; the error page is i18n-only (no GQL adapters, forms, or requesters).

## Related

- `docs/platforms/website/brand-identity-alignment.md` — Logo presets + brand pairings
- `docs/platforms/website/ui-foundation.md` — `Dims` corner radius tokens
- `.cursor/rules/website-page-title-helmet.mdc` — W14 Helmet title
- `.cursor/rules/website-logo-no-frame.mdc` — logo no-frame invariant
- `.cursor/rules/website-corner-radius-tokens.mdc` — corner radius token discipline
