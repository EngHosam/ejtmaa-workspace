---
name: cpanel-platform-governance
description: Enforces Ejtmaa cpanel platform governance for work under `cpanel/`. Use when creating, modifying, fixing, reviewing, or planning control-panel files, especially routes, pages, layouts, shared UI, `ui/base`, `resources/configs`, adapters, forms, auth, or GQL wiring.
---

# CPanel Platform Governance

## When to Use

Use this skill for any task that touches `cpanel/`, especially:

- `cpanel/src/resources/configs/*`
- `cpanel/src/app/services/*`
- `cpanel/src/app/ui/base/*`
- `cpanel/src/app/ui/components/*`
- `cpanel/src/app/ui/layouts/*`
- `cpanel/src/app/ui/pages/*`
- `cpanel/src/types/*`
- cpanel auth, routes, adapters, forms, socket, theme, or GQL work
- architectural review or planning for the control panel

## Instructions

### Required Reading

Read these before making or reviewing cpanel changes:

1. `docs/platforms/cpanel/overview.md`
2. `docs/platforms/cpanel/repository-inventory.md`
3. `docs/platforms/cpanel/component-structure.md`
4. `docs/platforms/cpanel/ui-foundation.md`
5. `docs/platforms/cpanel/data-flow-and-gql.md`
6. `docs/invariants/cpanel.md`
7. `.cursor/rules/cpanel-platform-governance.mdc`

### Core Working Rules

- Treat `cpanel/` as its own git repository.
- Distinguish strictly between:
  - current behavior proven by current files,
  - and governance for future implementation.
- Do not document or imply behavior that the current cpanel files do not prove.
- Do not treat placeholders as finished modules.

### Structure Rules

- `src/resources/` owns runtime configuration, translations, and shell resources.
- `src/app/services/` owns orchestration behavior.
- `src/app/ui/base/` owns framework-facing infrastructure.
- `src/app/ui/components/` owns shared product UI.
- `src/app/ui/layouts/` owns reusable shell layouts.
- `src/app/ui/pages/` owns route entry pages.
- `src/types/` owns shared type contracts.

Do not collapse these layers back into page-local or bootstrap-local code.

### UI Rules

- Use `src/app/ui/base/components/Utils.tsx` as the web UI foundation.
- Extend shared tokens through `src/resources/configs/theme.ts` and `src/resources/configs/utils.ts`.
- Prefer existing `Utils` props for visual/layout behavior before reaching for `baseCssStyle` or `cssStyle`; treat raw style escapes as the fallback only when the primitive API does not already cover the need.
- Keep product/business reusable UI out of `ui/base/`; place it in `ui/components/`.
- For `MainLayout` responsive shell work, keep one shared shell tree and let CSS/media queries own desktop-vs-mobile presentation; reserve React state for interactive shell state like drawer open/close, not layout-wide viewport branching.
- Reuse `website/` lessons at the architectural/usability level. Use the SSR web bootstrap (`MyPage` + `web-core`).

### Data Rules

- Treat adapters as the reserved read boundary.
- Treat forms/requesters as the reserved write boundary.
- Use `useShallowAdapter` / `useShallowForm` for route-scoped state with deterministic identities.
- Add shared `DATA_ADAPTERS` or `Forms` identifiers only for real shared boundaries.
- Treat GQL as the reserved future read architecture when backend-backed reads are implemented.

### Review Checklist

- [ ] The change respects current folder ownership.
- [ ] No current behavior is overstated beyond file evidence.
- [ ] Placeholder files are still described truthfully as placeholders when applicable.
- [ ] Product UI is not leaking into `ui/base/`.
- [ ] Route pages remain orchestration-oriented.
- [ ] Data/read/write ownership stays aligned with adapters/forms governance.
- [ ] Any material contract change updates the cpanel docs in the same task.

### Refusal Conditions

Stop and ask for clarification if:

- folder ownership is ambiguous,
- a proposed feature crosses multiple layers without a clear owner,
- current behavior is unclear and the task would require invention,
- backend contract assumptions are being made without evidence from current files or backend docs.
