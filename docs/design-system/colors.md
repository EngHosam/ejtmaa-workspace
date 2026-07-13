# Color Usage Guide (Ejtmaa)

Source of truth: `website/src/resources/configs/theme.ts`

## Layers

| Layer | Export | Use |
|---|---|---|
| 1 | `BrandColors` | Core navy/orange |
| 1.1 | `BrandScales` | Tint/shade ramps |
| 2 | `NeutralColors` | Surfaces, text neutrals |
| 3 | `FeedbackColors` | danger, warning, success, info |
| 4 | `SemanticColors` | primary, accent, semantic aliases |
| 5 | `UtilityColors` | scrims, focus rings, soft fills |

## Primary usage rules

- **Primary brand** → `SemanticColors.primary` (`#0B2057` navy)
- **CTA / highlight** → `SemanticColors.accent` (`#EC6901` orange)
- **Accent text on light** → `SemanticColors.accentText` (`#B94700`)

## Gradients

Prefer semantic gradient keys over raw hex:
- Auth/hero backgrounds → `SemanticGradients.hero`
- Primary buttons → `SemanticGradients.primary` or solid `SemanticColors.primary`
- Accent CTAs → `SemanticGradients.accent`

## Theme paths

Resolve colors through `ThemeMap` paths via `utils.ts`:
- Text: `base.text.content.default.primary`
- Surfaces: `surface.fill.container.default.card`
- Actions: `surface.fill.action.default.primary`

## Cross-platform

Both `website/` and `cpanel/` share this token system when using the same theme scaffold.

Full reference: `docs/design-color-system.md`
