# Website Brand Identity Alignment

Describes the current brand identity alignment of `website/` runtime URLs and consumer color usage against the approved Ejtmaa brand. Scope: `website/` only. `theme.ts` is brand authority and is not modified to satisfy consumers.

## URL configuration — `website/src/resources/configs/urls.ts`

`URLS(myInstance)` resolves Ejtmaa environments from `SHARED__TEST_MODE`:

| key | test mode (`SHARED__TEST_MODE=true`) | production |
|---|---|---|
| `ME_URL` | `http://192.168.1.10:3090` | `https://ejtmaa.live` |
| `BASE_URL` | `http://192.168.1.10:3206/cpanel` | `https://backend.ejtmaa.live/website` |
| `SOCKET_URL(namespace)` | `http://192.168.1.10:6662/${namespace}` | `https://backend.ejtmaa.live/${namespace}` |

Behavioral facts:

- Production `BASE_URL` uses the `/website` mount, matching W7 (`/website` only) and the website requester/GQL surface (`FORMS.CUSTOMER.R`, `DATA_ADAPTERS.GQL`, `CUSTOM.START` on `/website`).
- `ME_URL` production is `https://ejtmaa.live`.
- `SOCKET_URL` production namespaces are served from `https://backend.ejtmaa.live/${namespace}`.

## Semantic color foundation

`website/src/resources/configs/theme.ts` is brand authority (navy `#0B2057`, orange `#EC6901`, brand scales, neutral scale, feedback scales, utility colors, `ThemeMap.light` / `ThemeMap.dark`). `website/src/resources/configs/utils.ts` exposes `semanticColor` — a flat vocabulary of `ColorType` path strings pointing into `ThemeMap`.

Guarantees:

1. **All `semanticColor` paths are valid.** Each entry is typed `as ColorType`, and `ColorType` includes `ThemeMapPath` — a string-literal union derived from `ThemeMapType` via `FullNestedPaths`. A path that is not a real `ThemeMap` leaf is rejected by `tsc` at the `as` cast. `yarn type-check` passing is proof that every path resolves. The 48-key → `theme.ts` line map lives in `.cursor/skills/website-semantic-color-audit/SKILL.md`.
2. **Consumers use semantic tokens.** Every consumer under `website/src` accesses color through `semanticColor.<key>`; no consumer uses raw `"surface.fill...."` / `"base.text...."` path strings.
3. **Contrast pairing is correct.** Text/icon tokens are paired against the resolved surface fill (white/light action-text on primary/dark fills; navy `secondaryActionText` on the light `secondaryActionBackground` surface).

## Canonical consumer pairings (current)

| file | surface | text/icon |
|---|---|---|
| `website/src/app/ui/components/form/FormActionButton.tsx` | primary: `semanticColor.primaryActionBackground` (navy) | `semanticColor.primaryActionText` |
| `website/src/app/ui/components/form/FormActionButton.tsx` | neutral: `semanticColor.cardBackground` | `semanticColor.textPrimary` |
| `website/src/app/ui/components/ThemeModeSwitch.tsx` | single toggle button: `semanticColor.inputBackground` + `inputBorder`, corner `semanticDims.card.radius` | `semanticColor.iconPrimary` (moon when light → switch to dark; sun when dark → switch to light) |
| `website/src/app/ui/pages/UiMockup.tsx` `reviewed` badge | `semanticColor.secondaryActionBackground` (navy[50]) | `semanticColor.secondaryActionText` (navy) |
| `website/src/app/ui/pages/Error.tsx` primary CTA | `semanticColor.primaryActionBackground` (navy) (`actionBackground`) | `semanticColor.primaryActionText` |
| `website/src/app/ui/pages/Error.tsx` error code | `semanticColor.accentActionBackground` (orange) solid text | `semanticColor.accentActionBackground` |
| `website/src/app/ui/pages/Error.tsx` corner brackets | navy top-right (`primaryActionBackground`) / orange bottom-left (`accentActionBackground`) | `2px` border edges |

## Logo

`website/src/app/ui/components/Logo.tsx` renders the brand mark as a bare `Image` and swaps the source by color scheme: `dark_logo.png` on light scheme, `light_logo.png` on dark scheme. Assets live at `website/public/images/{dark,light}_logo.png`.

Size contract is the `preset` prop — the only way to size the logo. `LOGO_SIZES` sets **height only**; width is never hardcoded and is derived from the image's intrinsic ~3.4:1 aspect ratio (`width: auto`). No call-site width/height overrides.

| Preset | Height | Consumer |
|---|---|---|
| `header` | `4.875rem` | `Header.tsx` main variant brand slot |
| `drawer` | `7.7rem` | `Drawer.tsx` hero identity zone |
| `footer` | `3.25rem` | `Footer.tsx` brand slot |
| `hero` | `7.95rem` | `Error.tsx` centered hero card |

Add a new preset when a new size context appears (e.g. `hero` for the Error card) instead of overriding `h` at the call site.

**No frame around the logo.** Consumers render the bare `Logo` element with alignment only (`as_fs` / `as_c` / direct flex placement). The previous pill wrappers (`br` + `bg` + `p` + `crn={999}`) were removed from `Header.tsx`, `Footer.tsx`, `Drawer.tsx`, and `Error.tsx`. Do not reintroduce a border/background/padding container around `Logo`. Enforcement: `.cursor/rules/website-logo-no-frame.mdc`, invariant W45.

## Token discipline

- Prefer `semanticColor.<key>` over `@white` / `@<BaseColor>` hardcodes whenever a semantic token exists (W43).
- Do not add `semanticColor` tokens without a consumer (YAGNI). Do not edit `theme.ts` to satisfy a consumer.
- Enforcement: `.cursor/rules/website-semantic-color-token-discipline.mdc`. Audit procedure: `.cursor/skills/website-semantic-color-audit/SKILL.md`.

## Verification

- `yarn type-check` (`tsc --noEmit`) in `website/` is the sanctioned path-validity guard. No other tooling is introduced.

## Related

- `docs/platforms/website/ui-foundation.md` — `theme.ts` / `semanticColor` foundation
- `docs/platforms/website/shared-ui-and-shell.md` — drawer hero centering, theme toggle placement
- `docs/invariants/website.md` — W7 (mount), W10 (theme paths), W43, W44, W45 (logo), W46 (corner radius), W47 (no gradients)
- `.cursor/rules/website-semantic-color-token-discipline.mdc`
- `.cursor/rules/website-logo-no-frame.mdc`
- `.cursor/rules/website-no-gradients.mdc`
- `.cursor/skills/website-semantic-color-audit/SKILL.md`
