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
| `website/src/app/ui/components/form/FormActionButton.tsx` | primary: `segmentedSelected` navy gradient | `semanticColor.primaryActionText` |
| `website/src/app/ui/components/form/FormActionButton.tsx` | neutral: `semanticColor.cardBackground` | `semanticColor.textPrimary` |
| `website/src/app/ui/components/ThemeModeSwitch.tsx` | selected: `segmentedSelected` navy gradient | `semanticColor.primaryActionText` |
| `website/src/app/ui/components/ThemeModeSwitch.tsx` | unselected: transparent over `inputBackground` | `semanticColor.iconPrimary` |
| `website/src/app/ui/pages/UiMockup.tsx` `reviewed` badge | `semanticColor.secondaryActionBackground` (navy[50]) | `semanticColor.secondaryActionText` (navy) |

## Token discipline

- Prefer `semanticColor.<key>` over `@white` / `@<BaseColor>` hardcodes whenever a semantic token exists (W43).
- Do not add `semanticColor` tokens without a consumer (YAGNI). Do not edit `theme.ts` to satisfy a consumer.
- Enforcement: `.cursor/rules/website-semantic-color-token-discipline.mdc`. Audit procedure: `.cursor/skills/website-semantic-color-audit/SKILL.md`.

## Verification

- `yarn type-check` (`tsc --noEmit`) in `website/` is the sanctioned path-validity guard. No other tooling is introduced.

## Related

- `docs/platforms/website/ui-foundation.md` — `theme.ts` / `semanticColor` foundation
- `docs/invariants/website.md` — W7 (mount), W10 (theme paths), W43, W44
- `.cursor/rules/website-semantic-color-token-discipline.mdc`
- `.cursor/skills/website-semantic-color-audit/SKILL.md`
