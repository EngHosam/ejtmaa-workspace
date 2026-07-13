# Ejtmaa Design Color System

Source of truth: `website/src/resources/configs/theme.ts` (read-only reference for docs; do not edit code in docs-only tasks).

## Layer 1 — Brand colors

| Token | Hex |
|---|---|
| `BrandColors.navy` | `#0B2057` |
| `BrandColors.orange` | `#EC6901` |

## Layer 1.1 — Brand scales

### Navy scale

| Step | Hex |
|---|---|
| 50 | `#F0F4FA` |
| 100 | `#E2EAF5` |
| 200 | `#C7D5EA` |
| 300 | `#9AB3D7` |
| 400 | `#668BC2` |
| 500 | `#3D68A7` |
| 600 | `#244B88` |
| 700 | `#16366D` |
| 800 | `#0B2057` |
| 900 | `#071638` |
| 950 | `#040D22` |

### Orange scale

| Step | Hex |
|---|---|
| 50 | `#FFF6ED` |
| 100 | `#FFE9D5` |
| 200 | `#FFD0A8` |
| 300 | `#FFAE70` |
| 400 | `#FF8738` |
| 500 | `#EC6901` |
| 600 | `#D85A00` |
| 700 | `#B94700` |
| 800 | `#913800` |
| 900 | `#6E2C00` |
| 950 | `#401700` |

## Layer 2 — Neutral scale

| Step | Hex |
|---|---|
| 0 | `#FFFFFF` |
| 50 | `#FCFDFE` |
| 100 | `#F7F9FC` |
| 200 | `#EEF2F7` |
| 300 | `#DDE4EE` |
| 400 | `#C4CEDB` |
| 500 | `#97A3B4` |
| 600 | `#6B7788` |
| 700 | `#465366` |
| 800 | `#263244` |
| 900 | `#141D2B` |
| 950 | `#080E19` |

## Layer 3 — Feedback colors

| Role | default | strong | soft | onDark |
|---|---|---|---|---|
| danger | `#D92D20` | `#B42318` | `#FEE4E2` | `#FF7A70` |
| warning | `#9A6700` | `#7A5200` | `#FEF0C7` | `#FFD166` |
| success | `#14804A` | `#067647` | `#D1FADF` | `#5DD39E` |
| info | `#2563EB` | `#1D4ED8` | `#D1E9FF` | `#75A7FF` |

## Layer 4 — Semantic colors

| Token | Value |
|---|---|
| primary | `#0B2057` |
| secondary | `#071638` |
| accent | `#EC6901` |
| accentText | `#B94700` |
| danger | `#D92D20` |
| warning | `#9A6700` |
| success | `#14804A` |
| info | `#2563EB` |

## Layer 5 — Utility colors

| Token | Value |
|---|---|
| transparent | `rgba(0,0,0,0)` |
| scrim | `rgba(4,13,34,0.84)` |
| softScrim | `rgba(4,13,34,0.58)` |
| primarySoft | `rgba(11,32,87,0.08)` |
| primaryMuted | `rgba(11,32,87,0.14)` |
| primaryStrong | `rgba(11,32,87,0.22)` |
| accentSoft | `rgba(236,105,1,0.10)` |
| accentMuted | `rgba(236,105,1,0.16)` |
| accentStrong | `rgba(236,105,1,0.24)` |
| focusPrimary | `rgba(11,32,87,0.28)` |
| focusAccent | `rgba(236,105,1,0.30)` |
| focusDanger | `rgba(217,45,32,0.26)` |

## BaseColors (flat map)

Backward-compatible aliases: `primary`, `secondary`, `accent`, `accentText`, `danger`, `warning`, `success`, `info`, `light`, `dark`, `white`, `black`, `cloud`, `smoke`, `steel`, `space`, `graphite`, `arsenic`, `phantom`, `ink`, plus soft/muted/scrim variants.

## Gradients

| Key | Colors | Angle |
|---|---|---|
| surfaceLight | `#FFFFFF` → `#F7F9FC` | 135deg |
| surfaceDark | `#040D22` → `#141D2B` | 135deg |
| primary | `#16366D` → `#0B2057` | 135deg |
| primaryDark | `#0B2057` → `#040D22` | 135deg |
| accent | `#EC6901` → `#FF8738` | 135deg |
| accentDark | `#B94700` → `#EC6901` | 135deg |
| heroLight | `#FFFFFF` → navy 50 → orange 50 | 120deg |
| heroDark | `#040D22` → `#0B2057` | 120deg |

### SemanticGradients

| Key | light | dark |
|---|---|---|
| segmentedSelected | primary | primaryDark |
| surfaceDepth | surfaceLight | surfaceDark |
| hero | heroLight | heroDark |
| accentDepth | accent | accentDark |

## ThemeMap

`ThemeMap.light` and `ThemeMap.dark` define semantic paths under:
- `base.text`, `base.icon`, `base.border`, `base.focus`, `base.state`
- `surface.fill` (canvas, container, action, input, feedback, selection, table)
- `raised.fill.container`
- `overlay.fill.backdrop`

Use `utils.ts` path helpers in components — do not duplicate hex values.

## Fonts

| Key | Stack |
|---|---|
| arabic | IBM Plex Sans Arabic, Tajawal, system-ui |
| latin | Inter, system-ui |
| fallback | system-ui, Segoe UI, sans-serif |

## Dims (selected)

| Token | Value |
|---|---|
| corner | 16px |
| smallCorner | 10px |
| largeCorner | 24px |
| buttonHeight | 3rem |
| headerHeight | 80px |
| contentMaxWidth | 1280px |
| drawerWidth | 300px |

## Shadows

| Key | Value |
|---|---|
| xs | `0 1px 2px rgba(4,13,34,0.06)` |
| sm | `0 4px 12px rgba(4,13,34,0.08)` |
| md | `0 12px 30px rgba(4,13,34,0.12)` |
| lg | `0 24px 56px rgba(4,13,34,0.16)` |
| primaryGlow | `0 12px 32px rgba(11,32,87,0.22)` |
| accentGlow | `0 12px 32px rgba(236,105,1,0.22)` |

## Motion

| Token | Value |
|---|---|
| durationFast | 120ms |
| durationNormal | 200ms |
| durationSlow | 320ms |
| easeStandard | cubic-bezier(0.2, 0, 0, 1) |
| easeEmphasized | cubic-bezier(0.2, 0, 0, 1.2) |

## zIndex

| Layer | Value |
|---|---|
| loadable | 200 |
| toast | 180 |
| modals | 150 |
| header | 100 |
| menu | 80 |
| popover | 75 |
| drawer | 50 |
| drawerBackdrop | 40 |
| sticky | 30 |
