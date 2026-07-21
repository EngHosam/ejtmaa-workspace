---
name: website-semantic-color-audit
description: >-
  Audit website `semanticColor` (`website/src/resources/configs/utils.ts`) against
  `theme.ts` `ThemeMap`, verify path validity via the `ThemeMapPath` type guard, and
  fix consumer contrast pairings without editing `theme.ts`. Use when `theme.ts` or
  `semanticColor` changes, when a consumer color looks wrong, or when re-verifying
  brand identity alignment under `website/`.
---

# Website semanticColor audit

## When to Use

- After any change to `website/src/resources/configs/theme.ts` or `website/src/resources/configs/utils.ts`.
- When a website consumer color/contrast looks wrong or a `@white` / `@<BaseColor>` hardcoded shortcut is suspected.
- When re-verifying brand identity alignment (navy `#0B2057`, orange `#EC6901`) before ship.
- When reviewing whether a new `semanticColor` token is justified (YAGNI check).

## Instructions

1. **Do not edit `theme.ts`.** It is brand authority. Fixes go in `utils.ts` (remap/add only with a real consumer) or in the consumer (pairing fix).
2. **Inventory consumers.** Grep `semanticColor\.\w+` under `website/src` and list every file + the keys it uses. Confirm no consumer uses raw `"surface.fill...."` / `"base.text...."` path strings directly (all access goes through `semanticColor.<key>`).
3. **Verify path validity (automated).** Run `yarn type-check` (`tsc --noEmit`) in `website/`. Because each `semanticColor` entry is typed `as ColorType` and `ColorType` includes `ThemeMapPath` (a string-literal union derived from `ThemeMapType` via `FullNestedPaths`), an invalid path fails compilation. A passing `type-check` proves every path resolves.
4. **Line-map each key to its `theme.ts` leaf** (manual cross-check). Use the 48-key table below as the baseline; re-derive line numbers after any `theme.ts` change.
5. **Audit contrast pairings.** For each consumer `bg`/`clr` (or `bg`/`color`) pair, resolve `bg` to its `ThemeMap` leaf value in light and dark schemes and confirm the text/icon token is legible on that fill. Common defects:
   - `@white` / `textInverted` on a light fill (e.g. `secondaryActionBackground` = navy[50]) → invisible. Use the matching action-text token (`secondaryActionText`, etc.).
   - Hardcoded `@white` on a primary/accent fill → replace with `primaryActionText` / `accentActionText`.
6. **YAGNI on additions.** Only add a new `semanticColor` token when a real consumer needs a `ThemeMap` leaf not yet exposed. Dead tokens (no consumer) are forbidden. If no consumer needs a missing leaf, do not add it.
7. **No remap without evidence.** Do not remap an existing `semanticColor` entry to a different path unless the current path is provably wrong (resolves to a brand-inconsistent value or a non-existent leaf, which `type-check` already catches). Document any remap as a locked decision with `key → new path` and the reason.
8. **Verify no gradients (automated).** Grep `gradient|Gradient|linear-gradient|radial-gradient|conic-gradient` under `website/src`. It must return no matches. `theme.ts` exports no gradient API and no website style path may use a CSS gradient (including decorative overlays, clipped text, and scrollbar thumbs). Any hit is a defect: replace with a solid `semanticColor.*` token via `getColor` / Utils props. See `.cursor/rules/website-no-gradients.mdc`.
9. **Verify.** Re-run `yarn type-check`. Do not invent new test/tooling.

## Consumer inventory baseline (22 files)

`resources/configs/utils.ts`; `pages/UiMockup.tsx`; `pages/Error.tsx`; `pages/Customer.tsx`; `layouts/MainLayout.tsx`; `layouts/BasicLayout.tsx`; `components/form/FormTextField.tsx`; `components/form/FormInputWrapper.tsx`; `components/form/FormActionButton.tsx`; `components/customer/CustomersTable.tsx`; `components/customer/CustomerStatsSection.tsx`; `components/customer/CustomerIdentityCell.tsx`; `components/auth/AuthPageShell.tsx`; `components/Toast.tsx`; `components/ThemeModeSwitch.tsx`; `components/MicrobandCredit.tsx`; `components/Header.tsx`; `components/GlobalLoadable.tsx`; `components/Footer.tsx`; `components/Drawer.tsx`; `components/DataTable.tsx`.

## 52-key → theme.ts line map (baseline)

The four `stateSoft*` keys pair the soft feedback fill (chip/badge background) with the matching `state*` foreground (icon + text); both auto-resolve per scheme. Consumer: `components/customer/meetings/MeetingMetaChips.tsx`.


| semanticColor key | path | theme.ts line |
|---|---|---|
| pageBackground | surface.fill.container.default.page | 411 |
| canvasBackground | surface.fill.canvas.default.primary | 402 |
| cardBackground | surface.fill.container.default.card | 412 |
| elevatedBackground | surface.fill.container.default.elevated | 413 |
| navigationBackground | surface.fill.container.default.navigation | 414 |
| headerBackground | surface.fill.container.default.header | 415 |
| footerBackground | surface.fill.container.default.footer | 416 |
| drawerBackground | raised.fill.container.default.drawer | 496 |
| inputBackground | surface.fill.input.default.primary | 453 |
| inputMutedBackground | surface.fill.input.default.secondary | 454 |
| tableToolbarBackground | surface.fill.table.default.toolbar | 486 |
| tableHeaderBackground | surface.fill.table.default.header | 481 |
| tableRowOddBackground | surface.fill.table.default.rowOdd | 482 |
| tableRowEvenBackground | surface.fill.table.default.rowEven | 483 |
| tableRowHoverBackground | surface.fill.table.default.rowHover | 484 |
| tableRowSelectedBackground | surface.fill.table.default.rowSelected | 485 |
| primaryActionBackground | surface.fill.action.default.primary | 423 |
| secondaryActionBackground | surface.fill.action.default.secondary | 424 |
| accentActionBackground | surface.fill.action.default.accent | 425 |
| neutralActionBackground | surface.fill.action.default.neutral | 427 |
| dangerActionBackground | surface.fill.action.default.danger | 426 |
| primaryActionText | base.text.action.default.fill.primary | 219 |
| secondaryActionText | base.text.action.default.fill.secondary | 220 |
| accentActionText | base.text.action.default.fill.accent | 221 |
| neutralActionText | base.text.action.default.fill.neutral | 223 |
| divider | base.border.divider.default.primary | 321 |
| subtleDivider | base.border.divider.default.secondary | 322 |
| shellBorder | base.border.shell.default.card | 347 |
| navigationBorder | base.border.shell.default.navigation | 346 |
| footerBorder | base.border.shell.default.footer | 348 |
| inputBorder | base.border.input.default.primary | 328 |
| inputFocusBorder | base.border.input.default.focus | 330 |
| tableBorder | base.border.table.default.primary | 354 |
| backdrop | overlay.fill.backdrop.default.scrim | 507 |
| softBackdrop | overlay.fill.backdrop.default.soft | 508 |
| textPrimary | base.text.content.default.primary | 208 |
| textSecondary | base.text.content.default.secondary | 209 |
| textTertiary | base.text.content.default.tertiary | 210 |
| textInverted | base.text.content.default.inverted | 211 |
| textAccent | base.text.content.default.accent | 212 |
| iconPrimary | base.icon.content.default.primary | 276 |
| iconSecondary | base.icon.content.default.secondary | 277 |
| iconAccent | base.icon.content.default.accent | 279 |
| stateError | base.state.feedback.default.error | 371 |
| stateWarning | base.state.feedback.default.warning | 372 |
| stateSuccess | base.state.feedback.default.success | 373 |
| stateInfo | base.state.feedback.default.info | 374 |
| stateSoftError | base.state.feedback.soft.error | 377 |
| stateSoftWarning | base.state.feedback.soft.warning | 378 |
| stateSoftSuccess | base.state.feedback.soft.success | 379 |
| stateSoftInfo | base.state.feedback.soft.info | 380 |
| stateDisabled | base.state.interaction.default.disabled | 391 |

## Related

- `docs/platforms/website/brand-identity-alignment.md` — shipped outcome
- `docs/platforms/website/ui-foundation.md` — foundation
- `docs/design-color-system.md` § Solid colors only — no-gradients policy + traceability
- `docs/invariants/website.md` W10, W43, W44, W47
- `.cursor/rules/website-semantic-color-token-discipline.mdc` — enforcement rule
- `.cursor/rules/website-no-gradients.mdc` — no-gradients enforcement rule
