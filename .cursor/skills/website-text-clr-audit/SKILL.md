---
name: website-text-clr-audit
description: Audit website Text/Icn rendered on colored (action/section/custom) backgrounds for missing explicit clr that pairs with the fill, and verify baseCssStyle vs cssStyle precedence on reusable components. Use when a colored control/banner/section shows wrong text color, when reviewing reusable components for the consumer escape hatch, or after touching Utils Text/Box clr behavior.
disable-model-invocation: true
---

# Website Text/Clr Audit

Audits two recurring defect classes on `website/`:

1. `Text`/`Icn` on a colored fill missing an explicit `clr` → renders dark ink (e.g. dark text on orange).
2. `baseCssStyle` used where `cssStyle` was needed (or vice-versa), or a reusable component exposing the wrong escape hatch.

## Background (why these break)

- `Text` forces `clr={defaultTextColor}` (`textPrimary`, dark) and does NOT inherit the parent's `clr`. A bare `<Text>` on an orange/navy fill renders dark.
- In `Box`'s `css()`, `baseCssStyle` is overridden by shorthand props; `cssStyle` overrides them. See `.cursor/rules/website-utils-style-prop-precedence.mdc`.

## Audit steps

1. **Find colored containers.** Grep for the action/section fills:
   - `accentActionBackground` (orange), `primaryActionBackground` (navy), `sectionAccentBackground` (orange tint), `secondaryActionBackground`, `neutralActionBackground`.
2. **For each colored container, inspect every child `Text` and `Icn`.** Each MUST have:
   - an explicit `clr={semanticColor.<actionText>}` pairing with the fill, OR
   - a `cssStyle={{color: ...}}` override.
   - A bare `<Text>` / `<Icn>` with no `clr` on a colored fill is a defect.
3. **Pairing table:**
   - orange (`accentActionBackground`) → `accentActionText` (white). Never dark.
   - navy (`primaryActionBackground`) → `primaryActionText` (white).
   - secondary (`secondaryActionBackground`, light navy) → `secondaryActionText` (navy, dark).
   - neutral (`neutralActionBackground`, light grey) → `neutralActionText` (ink, dark).
   - orange tint (`sectionAccentBackground`) → do NOT put dark `textPrimary`/`textSecondary` on it; switch the surface to `cardBackground`/`sectionBrandBackground`, or use white.
4. **Reusable component escape hatch.** For any `CoreComponent` wrapper (like `Layout`/`Page`/`Form`), confirm it exposes `cssStyle` to consumers and forwards `baseCssStyle` only internally — not the reverse.
5. **Verify** with `yarn type-check` (paths resolve) and a grep pass: `rg "bg=\{semanticColor\.(accent|primary|secondary|neutral)ActionBackground|sectionAccentBackground"` then read each block's children.

## Report format

List defects as:

```
<file>:<line> — <Text|Icn> on <fill> missing clr → suggested <token>
```

Mark reusable-component escape-hatch issues separately:

```
<file>:<line> — reusable <Name> exposes baseCssStyle (should expose cssStyle)
```

## Related

- `.cursor/rules/website-text-clr-on-colored-bg.mdc`
- `.cursor/rules/website-utils-style-prop-precedence.mdc`
- `.cursor/rules/website-semantic-color-token-discipline.mdc`
