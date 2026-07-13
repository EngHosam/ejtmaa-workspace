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

## Gradients

Use `SemanticGradients` helpers:
- `primary` / `accent` — action and hero surfaces
- `hero` — landing/auth hero backgrounds
- `surfaceDepth` — card/section depth

```ts
getSemanticGradient("primary", colorScheme)
getGradientBackground(gradient)
```

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
- Pills/circles use `crn={999}` (the `rem()` helper in `Utils.tsx` passes numbers through as `Nrem`, so `999` overflows any element and renders as a pill).
- Do not hardcode `crn="1.6rem"` / `crn="1rem"` / similar rem literals for card/button corners — use `semanticDims.card.radius` or `Dims.*`. The `Error` page was the canonical fix site for this.
- Per-corner radius uses `crn_tr` / `crn_tl` / `crn_br` / `crn_bl` (not `br_tr`; `br_t`/`br_r`/`br_b`/`br_l` are border edges, not radii).

Enforcement: `.cursor/rules/website-corner-radius-tokens.mdc`, invariant W46.

## RTL

Website supports ar/en with cookie+reload locale switch.
Icon directional flip: `.cursor/rules/website-icon-rtl-flip.mdc`.

## Theme sharing across frontends

`cpanel/` uses the same theme family as `website/`. Reference `website/src/resources/configs/theme.ts` as brand authority when `cpanel/src/resources/configs/theme.ts` is absent from the checkout.

## Related

- `docs/design-color-system.md` — full token reference
- `docs/invariants/website.md` — theme invariants
