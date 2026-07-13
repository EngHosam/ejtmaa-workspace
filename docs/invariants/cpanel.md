# Ejtmaa CPanel Invariants

## Purpose

This file defines the non-negotiable implementation invariants for the control-panel frontend under `cpanel/`.
It complements the backend and website invariants and exists to keep cpanel growth disciplined while the repository is still foundation-heavy and product-light.

Interpretation rule:
- invariants below define what future work must respect,
- they must not be misread as proof that every reserved scaffold surface is already fully implemented today,
- when `cpanel/` is not checked out or modules are still landing, treat route and shell references as contract targets rather than live inventory proof.

## C1. Repository Ownership Invariant

`cpanel/` is its own git repository inside the workspace.
Git operations for cpanel files must run from `cpanel/`, not from `backend/`, `website/`, or by assuming the workspace root owns those files.

Rationale:
- Preserve correct repo-local history, diff, and review behavior.

## C2. SSR Bootstrap Authority Invariant

The control panel must keep its runtime entry flow centered on the existing SSR stack:
1. `src/client/*`
2. `src/server/*`
3. `src/resources/configs/web-core.ts`
4. `src/app/ui/base/core/MyApp.tsx`
5. `src/app/ui/base/core/MyHtml.tsx`

Ad-hoc bootstrap paths that bypass this chain are forbidden unless explicitly approved and documented.

Rationale:
- Keep one deterministic runtime composition model while the platform is still forming.

## C3. Folder Ownership Invariant

The current cpanel folder split is architectural and must be respected:
1. `src/resources/` owns runtime configuration, translations, and shell resources.
2. `src/app/services/` owns cpanel behavior orchestration.
3. `src/app/ui/base/` owns framework-facing UI infrastructure.
4. `src/app/ui/components/` owns shared project UI.
5. `src/app/ui/layouts/` owns shared shell/page layout composition.
6. `src/app/ui/pages/` owns route entry pages.
7. `src/types/` owns shared type contracts.

Feature work must not collapse these layers back into page-local or bootstrap-local code.

Rationale:
- Keep expansion predictable and avoid structural drift from the first implementation wave.

## C4. UI Foundation Invariant

`cpanel/src/app/ui/base/components/Utils.tsx` is the mandatory UI foundation for control-panel web surfaces.
Shared visual decisions must extend the current UI foundation through:
1. `Utils.tsx`
2. `src/resources/configs/theme.ts` (when present in cpanel checkout; brand reference: `website/src/resources/configs/theme.ts`)
3. `src/resources/configs/utils.ts`
4. shared project components under `src/app/ui/components/`

Unstructured raw markup/CSS duplication across pages is forbidden by default when the behavior belongs to shared UI language.

Rationale:
- Preserve one visual and structural language for the control panel.

## C5. Base-vs-Project UI Invariant

`src/app/ui/base/` is for infrastructure and framework binding only.
Business-facing reusable UI belongs in `src/app/ui/components/`, not in `base/`.

Examples of what should stay out of `base/`:
- business cards,
- tables,
- filters,
- dashboard widgets,
- settings sections,
- feature-specific panels.

Rationale:
- Prevent the infrastructure layer from becoming a dumping ground for product UI.

## C6. Page Orchestration Invariant

Route pages under `src/app/ui/pages/` should remain orchestration-oriented:
- choose hooks,
- compose layouts,
- wire adapters/forms/services,
- render shared UI.

Pages should not become the only place where reusable visual sections or transport plumbing live.

Rationale:
- Keep route entry files understandable and reusable UI extractable.

### C6A. Section-Level Extraction Invariant

When a route grows beyond a small orchestration file:
- extract reusable or large route sections into `src/app/ui/components/`,
- keep the route page thin,
- but do not fragment one module into file-per-private-helper when those helpers are only used inside one section.

Preferred granularity for the current cpanel style:
- route page -> orchestration only,
- section component file -> reusable within the route/module family,
- micro helper -> inline inside the owning section file unless reused elsewhere.

Rationale:
- Preserve the adopted cpanel component-organization style without route bloat or meaningless micro-files.

## C7. Shared Lesson Reuse Invariant

The control panel must reuse the mature **organizational and usability lessons** from `website/` where the concepts overlap:
- one shared UI foundation,
- one infrastructure layer,
- reads through adapters,
- writes through forms/requesters,
- predictable loading/busy/error/empty behavior,
- route files as orchestration layers.

Cpanel uses the SSR/web bootstrap (`MyPage` + `web-core`) while reusing shared UI discipline from `website/` at the architectural level.

## C8. Backend Surface Coupling Invariant

Control-panel frontend integration must target the backend `/cpanel` surface only through documented configuration authorities.

Current authorities:
1. `src/resources/configs/axios/api.ts`
2. `src/resources/configs/urls.ts`
3. `src/types/requesters/requesters.cpanel.ts`
4. backend contract docs under `docs/platforms/backend/contracts/http-and-requesters.md`

If backend requester subs, GraphQL schema inputs, or base URLs change, the cpanel configuration and docs must be updated in the same task.

Rationale:
- Prevent silent drift between frontend wiring and backend ownership.

## C9. Read Data Invariant

Backend-owned reads should flow through the adapter layer.

Current foundational authorities:
1. `src/resources/configs/store/data-adapters.ts`
2. `src/app/ui/base/components/Adapter.tsx`
3. `src/app/ui/base/hooks/useAdapter.tsx`
4. `src/app/ui/base/hooks/useShallowAdapter.tsx`

Route pages should not default to ad-hoc page-local transport code when a data-adapter boundary is the correct architecture.

Rationale:
- Keep read ownership explicit and aligned with the shared project pattern.

## C10. Write Action Invariant

Backend-owned writes should flow through the form/requester layer.

Current foundational authorities:
1. `src/resources/configs/store/forms.ts`
2. `src/app/ui/base/components/Form.tsx`
3. `src/app/ui/base/hooks/useForm.tsx`
4. `src/app/ui/base/hooks/useShallowForm.tsx`

Route pages should not default to raw requester posting when a form boundary is the correct architecture.

Rationale:
- Keep write ownership explicit and prevent duplicated transport/form state logic.

## C11. GQL Read Contract Invariant

The reserved read architecture for control-panel backend data is GQL through the adapter layer.

Current evidence:
1. `/data_adapters/gql` exists in `src/resources/configs/axios/api.ts`
2. `graphql.config.yml` points at `src/types/gql/definitions/{base,supervisor}.graphql`
3. copied/generated mirrors now exist under `src/types/gql/gql-types/{base,supervisor}.ts`

Future read work should extend this path instead of inventing a second read architecture.

Rationale:
- Preserve one future data contract for supervisor-facing reads.

### C11A. GQL Mirror Sync Invariant

For the cpanel/backend GraphQL mirror workflow:
1. Backend GraphQL files remain the source of truth.
2. CPanel mirrors both:
   - `src/types/gql/definitions/*.graphql`
   - `src/types/gql/gql-types/*.ts`
3. Mirror sync must happen by command-based copy, not by hand-editing copied mirror content.
4. Local `graphql.config.yml` files belong only in UI subtrees that actually host GraphQL-aware code.

Rationale:
- Prevent contract drift between backend-owned GraphQL source files and cpanel-consumed mirrors/tooling surfaces.

### C11B. Role Mirror Boundary Invariant

GraphQL mirrors must follow frontend ownership:
1. `cpanel/` mirrors `supervisor` because it is the supervisor client.
2. `website/` mirrors `customer` because it is the customer client.
3. Mirror only the role contracts consumed by each frontend.

Rationale:
- Prevent cross-role contract leakage and wrong-client ownership drift.

## C12. Shared Registry Discipline Invariant

`DATA_ADAPTERS` and `Forms` are for real shared boundaries only.
Do not add global shared identifiers for every route-private screen or modal by default.

Use deterministic `useShallowAdapter` / `useShallowForm` identities when the state is screen-scoped.

Rationale:
- Prevent global registries from becoming noisy and semantically meaningless.

## C13. Supervisor Actor Invariant

The current typed cpanel actor model is supervisor-only.
New control-panel surfaces must not invent customer/visitor page ownership inside `cpanel/` unless product scope explicitly expands and the docs are updated in the same task.

Rationale:
- Keep the control panel aligned with the backend `/cpanel` actor boundary.

## C14. Localization Baseline Invariant

The current cpanel localization baseline is:
1. Arabic-only locale wiring
2. RTL enabled through web-core localization config

Adding additional locales is allowed later, but work must extend the existing localization path instead of hardcoding mixed-language UI literals across pages/components.

Rationale:
- Preserve one consistent direction/localization model.

## C15. Placeholder Discipline Invariant

Current foundation markers (starter registries for modules still landing):
- `Forms.FORM1` where a module has not yet bound a named form registry entry
- `DATA_ADAPTERS.ADAPTER1` where a module has not yet bound a named adapter registry entry

Future work must replace or formalize these placeholders explicitly.
Treating them as production-complete behavior is forbidden.

Rationale:
- Prevent false assumptions while starter registry entries remain in place.

## C16. Start Contract Invariant

The long-term server-start truth for cpanel remains the backend-driven `/custom/start` flow.
Temporary local shortcuts may exist during scaffolding, but they must remain visibly temporary and should not become the undocumented steady-state behavior.

Rationale:
- Preserve a single future hydration/startup contract for auth and global data.

## C17. Documentation Parity Invariant

Any task that materially changes cpanel architecture, folder ownership, UI foundation, route layout, auth model, localization shape, forms/adapters, requester typing, GraphQL setup, or deployment/runtime flow must update:
1. `docs/platforms/cpanel/README.md`
2. `docs/platforms/cpanel/overview.md`
3. `docs/platforms/cpanel/repository-inventory.md`
4. `docs/platforms/cpanel/component-structure.md`
5. `docs/platforms/cpanel/ui-foundation.md`
6. `docs/platforms/cpanel/data-flow-and-gql.md`
7. `docs/platforms/cpanel/supervisor-admin-modules.md`
8. `docs/platforms/cpanel/customer-management.md`
9. `docs/platforms/cpanel/graphql-mirror-and-tooling.md`
10. relevant feature maintenance docs touched by the change
11. this invariant file when the invariant itself changes

Rationale:
- Keep cpanel documentation usable as a real build foundation for later implementation, debugging, and extension work.

## C18. Customer Stats KPI Invariant

Home `HomeStatCard` and list header `CustomerStatsSection` bind to root `customerStats.total_count`.
List pagination window counts come from `_Customer.total_count` on the `customers` query.
KPI surfaces use `customerStats.total_count` only.
