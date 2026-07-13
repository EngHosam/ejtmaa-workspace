# Website UI Foundation (Ejtmaa)

## Source of truth

Brand and theme tokens: `website/src/resources/configs/theme.ts`

Core brand:
- navy: `#0B2057`
- orange: `#EC6901`

Do not use hardcoded hex in components when a `theme.ts` semantic path exists.

## UI primitives

- `src/app/ui/base/components/Utils.tsx` — Box, Text, Button, Input, etc.
- `src/resources/configs/utils.ts` — `getThemePath(...)`, color scheme resolution

Compose through Utils props first; use `baseCssStyle` only when primitives lack the needed behavior.

## Semantic colors

From `theme.ts` `SemanticColors`:
- `primary` → navy `#0B2057`
- `accent` → orange `#EC6901`
- `secondary` → navy 900 `#071638`

## Theme map

`ThemeMap.light` / `ThemeMap.dark` provide semantic paths for:
- text, icon, border, surface, action, table, overlay, focus, state

Access via utils theme path helpers — do not hardcode hex in components when a path exists.

## Semantic color audit & type guard

`semanticColor` in `resources/configs/utils.ts` is a flat vocabulary of `ColorType` path strings pointing into `ThemeMap`. Two guarantees:

1. **Path validity is type-checked.** `ColorType` includes `ThemeMapPath` (a string-literal union derived from `ThemeMapType` via `FullNestedPaths`). A path string that is not a real `ThemeMap` leaf is rejected by `tsc` at the `as ColorType` cast. `yarn type-check` is the path-validity guard.
2. **Token over hardcoded shortcut.** Prefer `semanticColor.<key>` over `@white` / `@<BaseColor>` hardcodes, and pair text/icon color against the resolved surface fill (W43). Do not add `semanticColor` tokens without a consumer (YAGNI); do not edit `theme.ts` to satisfy consumers.

Full audit procedure and the 48-key → `theme.ts` line map: `.cursor/skills/website-semantic-color-audit/SKILL.md`. Shipped alignment outcome: `brand-identity-alignment.md`.

## Typography

- Arabic: IBM Plex Sans Arabic
- Latin: Inter

## Dimensions and motion

- `Dims` — corners, button heights, header/footer, content max-width
- `Shadows` — xs through lg, primaryGlow, accentGlow
- `Motion` — duration and easing tokens
- `zIndex` — drawer, header, modals, toast stacking

## Corner radius tokens

Corner radius is centralized in `Dims` (`website/src/resources/configs/theme.ts`). Current values:

| Token | Value | Use |
|---|---|---|
| `Dims.corner` | `8px` | Cards, inputs, buttons, drawer action rows — the default surface corner |
| `Dims.smallCorner` | `6px` | Modals (see `resources/emotion/styles/modals-manager.ts`) |
| `Dims.largeCorner` | `12px` | Large surfaces when a wider corner is intended |
| `Dims.pillCorner` | `999px` | Pills / circles (kept as-is; not a halvable corner) |

Consumption rules:

- Read the default card corner via `semanticDims.card.radius` (`resources/configs/utils.ts`, which points to `Dims.corner`), not by re-typing the literal.
- Pills/circles use `crn={999}` (the `rem()` helper in `Utils.tsx` passes numbers through as `Nrem`, so `999` overflows any element and renders as a pill). `crn={999}` is reserved for chips, accent bars, avatars, and blobs.
- Header icon buttons (`HeaderIconButton` in `Header.tsx`) and the hero/top-right toggles (`ThemeModeSwitch`, `LanguageSwitch`) use `semanticDims.card.radius` — rounded-square, not pills.
- Do not hardcode `crn="1.6rem"` / `crn="1rem"` / similar rem literals for card/button corners — use `semanticDims.card.radius` or `Dims.*`. The `Error` page was the canonical fix site for this.
- Per-corner radius uses `crn_tr` / `crn_tl` / `crn_br` / `crn_bl` (not `br_tr`; `br_t`/`br_r`/`br_b`/`br_l` are border edges, not radii).

Enforcement: `.cursor/rules/website-corner-radius-tokens.mdc`, invariant W46.

## Localization & RTL

- Locales are configured in `website/src/resources/configs/web-core.ts` `localization`: `locales: ["ar", "en"]`, `defaultLocale: "ar"`, `rtlLocales: ["ar"]`. `ar` is RTL, `en` is LTR.
- Translation modules live in `website/src/resources/translations/`: `ar.ts` (default locale, source of the `Tr` type that backs `useTranslator`) and `en.ts` (full mirror). The two files MUST stay key-mirrored — every key in `ar.ts` is present in `en.ts` with the same shape (W14 relies on this).
- Locale switching is cookie (`locale`) + full reload via `changeLocale(myInstance)(newLocale)` exported from `@my-ssr/web-core`; no client-side locale state, no partial reload. The current locale is read via `useMyInstance().getRouter().locale`.
- The sole UI surface for switching is `LanguageSwitch` (`website/src/app/ui/components/LanguageSwitch.tsx`): a single toggle button showing the **target** language letters (`EN` when current is `ar`, `ع` when current is `en`), placed beside `ThemeModeSwitch` in the header trailing cluster, the drawer hero identity zone, and `BasicLayout`'s top-right row.
- Icon directional flip: `.cursor/rules/website-icon-rtl-flip.mdc`.

Locale surface traceability (current state):

| Path | Role | Described in |
|---|---|---|
| `website/src/resources/configs/web-core.ts` | `localization`: `locales`/`defaultLocale`/`translations`/`rtlLocales` | `docs/platforms/website/ui-foundation.md` § Localization & RTL |
| `website/src/resources/translations/ar.ts` | default locale module; source of `Tr` type | `docs/platforms/website/ui-foundation.md` § Localization & RTL |
| `website/src/resources/translations/en.ts` | full English mirror of `ar.ts` | `docs/platforms/website/ui-foundation.md` § Localization & RTL |
| `website/src/app/ui/components/LanguageSwitch.tsx` | sole locale switch UI (target-language letters) | `docs/platforms/website/brand-identity-alignment.md` § Canonical consumer pairings; `docs/design-color-system.md` § Traceability |
| `website/src/app/ui/components/Header.tsx` | `LanguageSwitch` in trailing cluster; `HeaderIconButton` corner | `docs/platforms/website/shared-ui-and-shell.md` § Shared header behavior; `docs/platforms/website/brand-identity-alignment.md` § Canonical consumer pairings |
| `website/src/app/ui/components/Drawer.tsx` | `LanguageSwitch` paired with `ThemeModeSwitch` in hero | `docs/platforms/website/shared-ui-and-shell.md` § Side drawer |
| `website/src/app/ui/layouts/BasicLayout.tsx` | `LanguageSwitch` in top-right row | `docs/platforms/website/shared-ui-and-shell.md` § Shared primitives |
| `website/lib/tsconfig.tsbuildinfo` | generated TS build artifact | not narrated (generated) |

Enforcement: `.cursor/rules/website-locale-switch.mdc`, invariant W48.

## Theme sharing across frontends

`cpanel/` uses the same theme family as `website/`. Reference `website/src/resources/configs/theme.ts` as brand authority when `cpanel/src/resources/configs/theme.ts` is absent from the checkout.

## Related

- `docs/design-color-system.md` — full token reference
- `docs/invariants/website.md` — theme invariants
