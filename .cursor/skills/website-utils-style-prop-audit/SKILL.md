---
name: website-utils-style-prop-audit
description: >-
  Audits and fixes website Utils style-prop usage: baseCssStyle vs cssStyle merge
  precedence, shorthand-prop preference, rem() numeric values, and native
  `<button>` background neutralization. Use when authoring or reviewing website
  UI built on Utils (Box/Row/Col/Flex/Text/Icn), when a button/list/section
  shows a wrong (white) background in dark mode, or when refactoring style
  props across landing sections, header, footer, or drawer.
---

# Website Utils style-prop audit

## When to Use

- Authoring or reviewing any website UI built on `Utils` (`Box`/`Row`/`Col`/`Flex`/`Text`/`Icn`).
- A button, list item, or section shows a wrong (often white) background in dark mode.
- Refactoring `baseCssStyle`/`cssStyle`/shorthand props across a surface (landing sections, header, footer, drawer).
- A reusable component needs a consumer style escape hatch.

## Read first

- `.cursor/rules/website-utils-style-prop-precedence.mdc` (authoritative)
- `website/src/app/ui/base/components/Utils.tsx` — `Box` `css()` construction, `rem()`, `onClick` auto-cursor
- `website/src/resources/configs/theme.ts` — `ElementStyles.buttonReset`, `ElementStyles.inputReset`
- `.cursor/rules/website-text-clr-on-colored-bg.mdc`
- `.cursor/rules/website-semantic-color-token-discipline.mdc`

## Core invariants

1. **Merge order inside `Box.css()`** (later wins): reset → `baseCssStyle` → shorthand props → `cssStyle` → `disabledStyle`/`hidden`/`hideAt`/`showAt`.
2. **`baseCssStyle` is overridden by shorthands.** Use it ONLY for `...ElementStyles.buttonReset` or a genuine default a shorthand must override (e.g. `background: "transparent"` under a conditional `bg`). Nothing else.
3. **`cssStyle` overrides shorthands.** Put every regular no-shorthand value here: `:hover`, `@media`, `transition`, `transform`, `transformOrigin`, `cursor`, `overflowX/Y`, `boxShadow`, `display`, `flex`, `gridTemplateColumns`, `insetInlineStart`, `scrollMarginTop`, `lineHeight`, `letterSpacing`, `fontVariantNumeric`, `textTransform`, `fontFamily`, `listStyle`, `backgroundImage`/`Size`/`Position`, `paddingBlock`, `marginBlock`, logical `textAlign`, asymmetric `borderRadius`, `backdropFilter`, `pointerEvents`.
4. **Prefer a shorthand prop** over either layer when one exists: `fs`, `opc`, `maxW`, `minW`, `mt`, `crn`, `br`, `bg`, `clr`, `asp`, `lh` (note: `lh` forces `rem()`).
5. **`rem()`**: pass a number for rem values (`fs={0.8}`, not `fs={"0.8rem"}`); pass a string only for non-rem (`"100%"`, `"1 / 1"`, `"0.85rem 1.25rem"`). Unitless `lineHeight` (e.g. `1.3`) must live in `cssStyle` — `lh` would turn it into `1.3rem`.
6. **Native `<button>` bg**: `buttonReset` does NOT reset `background`. Every `As={"button"}` must neutralize the browser default — three patterns: always-filled → `bg={...}`; conditionally-filled (`bg={active?X:undefined}`) → `background:"transparent"` in `baseCssStyle` (so the `bg` shorthand can win for active); always-transparent (never filled) → `bg="@transparent"` shorthand (not `baseCssStyle`/`cssStyle` background).
7. **`cursor: "pointer"`** is auto-added when the `onClick` prop is set — do not duplicate it. Set `cursor` explicitly only for `extra.onClick` wiring or non-pointer cursors.
8. **Reusable components** expose `cssStyle` to consumers, never `baseCssStyle`.

## Audit workflow

For each file under review, run this pass. Use Grep with `pattern: "baseCssStyle=\{\{"` and `pattern: "As=\{?[\"']button[\"']\}?"` to find candidates.

1. **Every `baseCssStyle={{...}}` block** — for each property inside:
   - `...ElementStyles.buttonReset` → keep (base).
   - A default a shorthand must override (e.g. `background: "transparent"` under conditional `bg`) → keep (base).
   - Anything else (`:hover`, `@media`, `transition`, `transform`, `cursor`, `lineHeight`, `letterSpacing`, …) → move to `cssStyle`. If the element already has `cssStyle`, merge into it.
   - If after removal `baseCssStyle` would be empty, delete the prop entirely.
2. **Shorthand collisions** — any `baseCssStyle`/`cssStyle` property that has a shorthand (`fontSize`→`fs`, `opacity`→`opc`, `maxWidth`→`maxW`, `minWidth`→`minW`, `marginTop`→`mt`, `borderRadius`→`crn`, `border`→`br`, `background`→`bg`, `color`→`clr`, `aspectRatio`→`asp`) → replace with the shorthand prop.
3. **Numeric rem values** — any rem-based shorthand assigned a `"Xrem"` string → change to the number `X` (`fs={"0.8rem"}` → `fs={0.8}`, `mt={"-0.15rem"}` → `mt={-0.15}`).
4. **`As={"button"}` elements** — confirm each neutralizes the native background by pattern:
   - Always-filled → has `bg={...}`. OK.
   - Conditionally-filled (`bg={active?X:undefined}`) → has `background: "transparent"` in `baseCssStyle` (NOT `cssStyle`, so the `bg` shorthand wins for active). Fix if missing — this is the dark-mode white-background bug.
   - Always-transparent (never filled) → has `bg="@transparent"` shorthand (NOT `background:"transparent"` in `baseCssStyle`/`cssStyle`). Fix if it uses the wrong layer.
5. **`cursor: "pointer"`** — if the element sets the `onClick` prop, remove the redundant `cursor`. Keep it only for `extra.onClick` or non-pointer cursors.
6. **`lineHeight` unitless multiplier** — must be in `cssStyle`, never the `lh` prop.

## Fix patterns

Conditional-fill button (active/inactive) — the FAQ IndexItem pattern:

```tsx
<Row
    As={"button"}
    bg={active ? semanticColor.sectionBrandBackground : undefined}
    baseCssStyle={{
        ...ElementStyles.buttonReset,
        background: "transparent"
    }}
    cssStyle={{
        textAlign: "start",
        cursor: "pointer",
        transition: "background-color 0.2s ease"
    }}
    extra={{onClick: onSelect}}
>
```

Outline/ghost button (never filled, no conditional `bg`) — transparent via the `bg` shorthand, not `baseCssStyle`/`cssStyle`:

```tsx
<Row
    As={"button"}
    br={[semanticColor.accentActionBackground, "1px"]}
    clr={semanticColor.accentActionBackground}
    bg="@transparent"
    baseCssStyle={{...ElementStyles.buttonReset}}
    cssStyle={{
        cursor: "pointer",
        transition: "background-color 0.18s ease",
        ":hover": {backgroundColor: accent + "14"}
    }}
    extra={{type: "button", onClick}}
>
```

Responsive `@media` block — belongs in `cssStyle`:

```tsx
<Col
    w={"100%"}
    cssStyle={{
        [landingShellDesktopMedia]: {
            display: "grid",
            gridTemplateColumns: "0.82fr 1.18fr"
        }
    }}
>
```

## Compliance checklist

- [ ] No `baseCssStyle` property remains except `...ElementStyles.buttonReset` or a shorthand-overridable default.
- [ ] All `:hover`/`@media`/`transition`/`transform`/`cursor`/`lineHeight`/`letterSpacing`/… live in `cssStyle`.
- [ ] No `baseCssStyle`/`cssStyle` property duplicates a shorthand; shorthands used instead.
- [ ] No rem-based shorthand holds a `"Xrem"` string — all are numbers.
- [ ] Every `As={"button"}` neutralizes its native background: always-filled → `bg={...}`; conditionally-filled → `background:"transparent"` in `baseCssStyle` + conditional `bg` shorthand; always-transparent → `bg="@transparent"` shorthand.
- [ ] No redundant `cursor: "pointer"` where `onClick` prop is set.
- [ ] Unitless `lineHeight` is in `cssStyle`, not `lh`.
- [ ] Reusable components expose `cssStyle`, not `baseCssStyle`.
- [ ] `yarn type-check` passes in `website/` after changes.

## Scope

- Applies to all website UI: `ui/components/**`, `ui/pages/**`, `ui/layouts/**`, and `ui/base/components/Utils.tsx` consumers.
- Do NOT refactor shared general components outside the task's surface (Drawer, DataTable, Toast, FormActionButton, …) unless explicitly in scope — preserve their observable behavior and respect platform boundaries.
