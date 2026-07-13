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

## Typography

- Arabic: IBM Plex Sans Arabic
- Latin: Inter

## Dimensions and motion

- `Dims` — corners, button heights, header/footer, content max-width
- `Shadows` — xs through lg, primaryGlow, accentGlow
- `Motion` — duration and easing tokens
- `zIndex` — drawer, header, modals, toast stacking

## RTL

Website supports ar/en with cookie+reload locale switch.
Icon directional flip: `.cursor/rules/website-icon-rtl-flip.mdc`.

## Theme sharing across frontends

`cpanel/` uses the same theme family as `website/`. Reference `website/src/resources/configs/theme.ts` as brand authority when `cpanel/src/resources/configs/theme.ts` is absent from the checkout.

## Related

- `docs/design-color-system.md` — full token reference
- `docs/invariants/website.md` — theme invariants
