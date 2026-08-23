# Cpanel UI Foundation (Ejtmaa)

Supervisor cpanel contract. Brand and Utils usage apply in the checked-in `cpanel/` repository.

## Brand authority

Ejtmaa brand tokens are defined in `website/src/resources/configs/theme.ts`:
- navy: `#0B2057`
- orange: `#EC6901`

When `cpanel/src/resources/configs/theme.ts` is absent from the checkout, treat `website/src/resources/configs/theme.ts` as the reference for brand colors and semantic paths.

## UI primitives

Same scaffold family as `website/`:
- `ui/base/components/Utils.tsx`
- `resources/configs/utils.ts`
- `resources/configs/theme.ts` (when present in cpanel checkout)

Compose through Utils props first. No parallel styling system.

## Semantic usage

- Primary actions: `SemanticColors.primary` (navy)
- Accent highlights: `SemanticColors.accent` (orange)

Use `theme.ts` tokens — do not hardcode hex when a semantic path exists.

## Supervisor context

Cpanel serves the Supervisor actor on `/cpanel`. Theme usage follows the same token map. Current layout is the MAIN admin shell plus login/error BASIC pages. Customer-management screens are not part of this bootstrap.

## Related

- `docs/platforms/website/ui-foundation.md` — detailed theme reference
- `docs/design-color-system.md` — full token listing
- `docs/platforms/cpanel/overview.md`
