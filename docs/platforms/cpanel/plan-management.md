# CPanel Plan Management (باقة)

Supervisor **plan** catalog: ResultLane directory plus multi-path create/update/delete form. Arabic UI: **باقة**. Writes are requester `plan`, not GraphQL mutations.

Backend reads: `docs/platforms/backend/contracts/supervisor-catalog-and-home.md` §3. Writes: same page §3.1. List DNA: `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`. Lanes: `supervisor-state-lanes.md`. Shared fields: `supervisor-shared-ui.md`.

## 1) Routes

| Identify | Path | Notes |
|---|---|---|
| `SupervisorPlans` | `/supervisor/plans` | List **before** form |
| `SupervisorPlanForm` | create `/supervisor/plans/form`; update `/supervisor/plans/form/:id` | `path: { create, update }`; `MPagesRoutes` `params.id?` |

`buildSupervisorPlanFormHref(sub, id?)` in `cpanel/src/resources/configs/supervisor/formRoute.ts` returns `{ identify: "SupervisorPlanForm", sub, params }`.

Drawer: `FiLayers`, `alwaysAvailable`, after meetings.

## 2) List

`useSupervisorPlans`: mount-private `"supervisor-plans"`, inherit `SUPERVISOR_GQL`, `listable: "plans"`, history search, `planStats` (`total_count`, `active_count`) on `Stats`.

`SupervisorPlanCard`: name, `SarAmount` monthly/yearly, status label. Add/edit href via `buildSupervisorPlanFormHref`. Delete is **not** on the card.

## 3) Form (`useSupervisorPlanForm`)

`Forms.SUPERVISOR_PLAN` → `API.FORMS.SUPERVISOR.R("plan")`.

| Mode | Slot | `didEntered` | `removeOnExit` | Defaults |
|---|---|---|---|---|
| create | extra `formIdentify: "supervisor-plan-form-create"` | no read | false | `status: "ACTIVE"`, `sort_order: 0` |
| update | inherited identify only | `sub: "read"` with `values.plan = id` | true | — |

Save: `d.send({ sub: formType })`. Success: create `d.reset()` then `nav.back()`; update `nav.back()`.

Delete: only if `_Me.canDeletePlan(planId).value === "CAN"` (separate inherit `SUPERVISOR_GQL` query). UI `confirm(..., "danger")` then `sub: "delete"`. Ability `CANNOT_DELETE_USED` when any `Subscription.plan_id` matches — button hidden. Requester still enforces the same check. Destroy is `force: true` (hard delete).

Fields: `FormMultiLangField` name/description; numbers for prices/limits/`sort_order`; `FormChoiceField` ACTIVE/DISABLED. Overlay: `FormStateLane`; heading + save/delete **outside** the lane.

## 4) Backend write contract (must match)

`PlanRequester` `@requester("plan")` `@sub(["cpanel"], ["supervisor"])`: `read` / `create` / `update` / `delete`. Each sub calls `supervisor.can("PLAN", { sub, plan?, throwMode: true })`.

Joi: `name` required `isMultiLang`; `description` `isMultiLang(joi, 4, true)` (null / empty pair → null); prices `number().min(1)`; limits optional integer `min(1)` or null; `status` ACTIVE|DISABLED.

Cpanel sibling requesters in this slice call `startTransaction()` **without** `"join_current"` (`SupervisorRequester`, `CustomerRequester`, `PlatformSettingsRequester`, `WebsiteSettingsRequester`).

## 5) Traceability

| Path | Role |
|---|---|
| `cpanel/src/app/ui/components/supervisor/plans/useSupervisorPlans.ts` | List adapter |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlansScreen.tsx` | List UI |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlanCard.tsx` | Card + skeleton |
| `cpanel/src/app/ui/components/supervisor/plans/useSupervisorPlanForm.ts` | Form + canDelete + confirm |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlanFormScreen.tsx` | Form UI |
| `cpanel/src/app/ui/pages/supervisor/SupervisorPlans.tsx` | Thin list |
| `cpanel/src/app/ui/pages/supervisor/SupervisorPlanForm.tsx` | Thin form |
| `cpanel/src/resources/configs/supervisor/formRoute.ts` | Href helper |
| `cpanel/src/resources/configs/store/forms.ts` | `SUPERVISOR_PLAN` |
| `backend/src/app/orchestrator/requesters/PlanRequester.ts` | Writes |
| `backend/src/app/orm/models/Supervisor.ts` | `Ability.PLAN` + `CANNOT_DELETE_USED` |
| `backend/src/app/gql/bridges/supervisor/PlanBridge.ts` | GQL reads |
| `backend/src/app/gql/bridges/supervisor/PlanStatsBridge.ts` | Stats extras |
| `backend/src/app/gql/bridges/supervisor/MeBridge.ts` | `canDeletePlan` |

## Related

- `docs/platforms/cpanel/customer-management.md`
- `docs/platforms/cpanel/supervisor-state-lanes.md`
- `docs/platforms/cpanel/supervisor-shared-ui.md`
- `.cursor/skills/website-confirm-modal/SKILL.md` (cpanel `confirm()` same helper)
