# Supervisor workspace change-set inventory

Exhaustive map of files that shipped the supervisor catalog, meetings, settings, and home work (customers directory already documented in `customer-management.md`). Nested git: `backend/`, `cpanel/`, workspace root. `website/` had no files in this change set.

Generated / command-copy paths are listed with rationale; they are not narrated line-by-line.

## 1) Workspace root (`docs/`, `.cursor/`)

| Path | Kind | Documented |
|---|---|---|
| `docs/platforms/backend/contracts/supervisor-catalog-and-home.md` | New contract | that page |
| `docs/platforms/cpanel/supervisor-home.md` | New | that page |
| `docs/platforms/cpanel/plan-management.md` | New | that page |
| `docs/platforms/cpanel/subscription-management.md` | New | that page |
| `docs/platforms/cpanel/meeting-management.md` | New | that page |
| `docs/platforms/cpanel/supervisor-settings.md` | New | that page |
| `docs/platforms/cpanel/supervisor-state-lanes.md` | New | that page |
| `docs/platforms/cpanel/supervisor-shared-ui.md` | New | that page |
| `docs/platforms/cpanel/supervisor-workspace-change-set.md` | This inventory | this page |
| `docs/platforms/cpanel/graphql-mirror-and-tooling.md` | Index | that page |
| `docs/platforms/cpanel/ui-foundation.md` | Index | that page |
| `docs/platforms/cpanel/route-registry-contract.md` | Routes | that page |
| `docs/platforms/backend/contracts/plan-domain.md` | Supervisor writes exist | that page |
| `docs/platforms/backend/contracts/subscription-domain.md` | Supervisor GQL exists | that page |
| `docs/platforms/backend/contracts/meeting-domain.md` | Supervisor directory exists | that page |
| `.cursor/rules/cpanel-platform-governance.mdc` | Platform governance | rule |
| `.cursor/rules/plan-catalog-domain.mdc` | Supervisor Plan GQL/writes | rule |
| `.cursor/rules/subscription-domain.mdc` | Supervisor Subscription GQL | rule |
| `.cursor/rules/backend-requester-whitespace-normalization.mdc` | `isMultiLang` optional | rule |
| `docs/platforms/cpanel/README.md` | Index | this tree |
| `docs/platforms/cpanel/overview.md` | Index | overview |
| `docs/platforms/cpanel/supervisor-admin-modules.md` | Index | that page |
| `docs/platforms/cpanel/repository-inventory.md` | Index | that page |
| `docs/platforms/cpanel/component-structure.md` | Index | that page |
| `docs/platforms/cpanel/data-flow-and-gql.md` | Index | that page |
| `docs/platforms/cpanel/flow-supervisor-shell.md` | Shell | that page |
| `docs/platforms/cpanel/customer-management.md` | Customers + drawer/KPI split | that page |
| `docs/invariants/cpanel.md` | C17 C18 | that file |
| `docs/platforms/backend/README.md` | Index | that page |
| `docs/platforms/backend/overview.md` | Bridges / ORM | that page |
| `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md` | Index | that page |
| `docs/platforms/backend/contracts/graphql-and-types.md` | Root queries | that page |
| `docs/platforms/backend/contracts/http-and-requesters.md` | Plan requester + visitor forms split | that page |
| `docs/platforms/backend/contracts/account-settings-requesters.md` | Points to supervisor settings | that page |
| `docs/platforms/backend/contracts/socket-event-mirroring.md` | `OnSupervisorEvent` | that page |
| `docs/platforms/backend/patterns/requesters-and-orchestration.md` | Inventory | that page |
| `docs/platforms/backend/modules/nodejs-socket-library.md` | Event list | that page |
| `docs/platforms/backend/modules/runtime-integrations.md` | Events | that page |
| `docs/platforms/backend/patterns/models-owners-abilities-security.md` | PLAN ability | that page |
| `docs/platforms/backend/playbooks/add-update-fix.md` | Playbook | that page |
| `docs/workspace-overview.md` | Workspace index | that page |
| `docs/README.md` | Map | that page |
| `.cursor/rules/cpanel-supervisor-home.mdc` | Home invariant | rule |
| `.cursor/skills/cpanel-supervisor-home/SKILL.md` | Home workflow | skill |
| `.cursor/rules/cpanel-supervisor-read-directory.mdc` | Directory invariant | rule |
| `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md` | Directory workflow | skill |
| `.cursor/rules/gql-association-auto-relations.mdc` | No invented lazy home relations | rule |
| `.cursor/skills/gql-schema-bridge-generator/SKILL.md` | Supervisor surfaces + HomeBridge template | skill |
| `backend` / `cpanel` (root git pointers) | Nested repo dirty flags | not product |

## 2) Backend

| Path | Kind | Documented |
|---|---|---|
| `backend/src/app/gql/definitions/supervisor.graphql` | SDL | catalog-and-home |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated types | catalog-and-home (no line-by-line) |
| `backend/src/app/gql/schemas/SupervisorSchema.ts` | Roots + registered bridges | catalog-and-home |
| `backend/src/app/gql/bridges/supervisor/HomeBridge.ts` | Home extras | catalog-and-home §6 |
| `backend/src/app/gql/bridges/supervisor/PlanBridge.ts` | Plan reads | catalog-and-home §3 |
| `backend/src/app/gql/bridges/supervisor/PlanStatsBridge.ts` | Plan stats | catalog-and-home §3 |
| `backend/src/app/gql/bridges/supervisor/SubscriptionBridge.ts` | Subscription reads | catalog-and-home §4 |
| `backend/src/app/gql/bridges/supervisor/SubscriptionStatsBridge.ts` | Subscription stats | catalog-and-home §4 |
| `backend/src/app/gql/bridges/supervisor/MeetingBridge.ts` | Meetings; `live_state` expect | catalog-and-home §5 |
| `backend/src/app/gql/bridges/supervisor/MeetingStatsBridge.ts` | Meeting stats | catalog-and-home §5 |
| `backend/src/app/gql/bridges/supervisor/MemberBridge.ts` | Nested chairperson | catalog-and-home §5 |
| `backend/src/app/gql/bridges/supervisor/MeBridge.ts` | `canDeletePlan` extra | catalog-and-home §3 |
| `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts` | `GetOneParent` includes Subscription/Organization | catalog-and-home §9 |
| `backend/src/app/gql/bridges/supervisor/OrganizationBridge.ts` | Nested parent types | organization-domain + catalog §9 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `OrganizationModel` | organization-domain |
| `backend/src/app/gql/bridges/customer/PlanBridge.ts` | `GetOneParent` includes `PlanModel` | plan-domain |
| `backend/src/app/orchestrator/requesters/PlanRequester.ts` | Plan CRUD | catalog-and-home §3; http-and-requesters |
| `backend/src/app/orchestrator/requesters/SupervisorRequester.ts` | Settings + `OnSupervisorEvent` | supervisor-settings |
| `backend/src/app/orchestrator/requesters/CustomerRequester.ts` | `startTransaction()` no `join_current` | catalog-and-home §3.1 |
| `backend/src/app/orchestrator/requesters/PlatformSettingsRequester.ts` | same | catalog-and-home §3.1 |
| `backend/src/app/orchestrator/requesters/WebsiteSettingsRequester.ts` | same | catalog-and-home §3.1 |
| `backend/requesters.cpanel.ts` | Registers `plan` | http-and-requesters |
| `backend/src/app/http/controllers/cpanel/forms/VisitorRequesterController.ts` | Visitor forms (`Authed()`) | http-and-requesters |
| `backend/src/app/http/routes/cpanel.ts` | Split global vs supervisor form routers | http-and-requesters |
| `backend/src/app/notify/events/OnSupervisorEvent.ts` | Socket `UPDATED` | socket-event-mirroring; supervisor-settings |
| `backend/src/app/types/Events.ts` | Event type | socket-event-mirroring |
| `backend/src/app/socket/controllers/ConnectionIOController.ts` | Supervisor room join | nodejs-socket-library |
| `backend/src/resources/configs/notify/index.ts` | Event registration | socket-event-mirroring |
| `backend/src/resources/consts/NotificationsConsts.ts` | Consts | socket-event-mirroring |
| `backend/src/app/orm/models/Supervisor.ts` | Ability surface used by PLAN / notify | models-owners-abilities-security |
| `backend/src/app/validation/joi_rules.ts` | `isMultiLang` optional + `isPlan` | models-owners; catalog-and-home §3.1 |

## 3) Cpanel (every uncommitted path)

| Path | Kind | Documented |
|---|---|---|
| `cpanel/lib/tsconfig.tsbuildinfo` | Generated `tsc` cache | not product |
| `cpanel/src/app/ui/components/FormStateLane.tsx` | Alias | supervisor-state-lanes |
| `cpanel/src/app/ui/components/PageStateLane.tsx` | Overlay host | supervisor-state-lanes |
| `cpanel/src/app/ui/components/Stats.tsx` | `href` / `note` | supervisor-shared-ui; C18 |
| `cpanel/src/app/ui/components/StatusChip.tsx` | Status pills | supervisor-shared-ui |
| `cpanel/src/app/ui/components/form/FormMultiLangField.tsx` | ar/en fields | supervisor-shared-ui |
| `cpanel/src/app/ui/components/form/FormTextField.tsx` | `disabled` | supervisor-shared-ui |
| `cpanel/src/app/ui/components/supervisor/SupervisorDrawer.tsx` | Tiles + Support | flow-supervisor-shell |
| `cpanel/src/app/ui/components/supervisor/customers/SupervisorCustomerScreen.tsx` | StatusChip + lane | customer-management |
| `cpanel/src/app/ui/components/supervisor/customers/useSupervisorCustomer.ts` | Overlay busy | supervisor-shared-ui §5 |
| `cpanel/src/app/ui/components/supervisor/home/SupervisorHomeBars.tsx` | Charts | supervisor-home |
| `cpanel/src/app/ui/components/supervisor/home/SupervisorHomeScreen.tsx` | Dashboard | supervisor-home |
| `cpanel/src/app/ui/components/supervisor/home/useSupervisorHome.ts` | Query + cap 5 | supervisor-home |
| `cpanel/src/app/ui/components/supervisor/hooks/useMe.tsx` | Socket reload | supervisor-settings |
| `cpanel/src/app/ui/components/supervisor/meetings/SupervisorMeetingCard.tsx` | Card | meeting-management |
| `cpanel/src/app/ui/components/supervisor/meetings/SupervisorMeetingScreen.tsx` | Detail | meeting-management |
| `cpanel/src/app/ui/components/supervisor/meetings/SupervisorMeetingsScreen.tsx` | List | meeting-management |
| `cpanel/src/app/ui/components/supervisor/meetings/useSupervisorMeeting.ts` | Detail hook | meeting-management |
| `cpanel/src/app/ui/components/supervisor/meetings/useSupervisorMeetings.ts` | List hook | meeting-management |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlanCard.tsx` | Card + `SarAmount` | plan-management |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlanFormScreen.tsx` | Form UI | plan-management |
| `cpanel/src/app/ui/components/supervisor/plans/SupervisorPlansScreen.tsx` | List UI | plan-management |
| `cpanel/src/app/ui/components/supervisor/plans/useSupervisorPlanForm.ts` | Form hook | plan-management |
| `cpanel/src/app/ui/components/supervisor/plans/useSupervisorPlans.ts` | List hook | plan-management |
| `cpanel/src/app/ui/components/supervisor/settings/SupervisorSettingsScreen.tsx` | Settings UI | supervisor-settings |
| `cpanel/src/app/ui/components/supervisor/settings/useSupervisorSettings.ts` | Settings hook | supervisor-settings |
| `cpanel/src/app/ui/components/supervisor/subscriptions/SupervisorSubscriptionCard.tsx` | Card | subscription-management |
| `cpanel/src/app/ui/components/supervisor/subscriptions/SupervisorSubscriptionScreen.tsx` | Detail | subscription-management |
| `cpanel/src/app/ui/components/supervisor/subscriptions/SupervisorSubscriptionsScreen.tsx` | List | subscription-management |
| `cpanel/src/app/ui/components/supervisor/subscriptions/useSupervisorSubscription.ts` | Detail hook | subscription-management |
| `cpanel/src/app/ui/components/supervisor/subscriptions/useSupervisorSubscriptions.ts` | List hook | subscription-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorMeeting.tsx` | Thin detail | meeting-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorMeetings.tsx` | Thin list | meeting-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorPlanForm.tsx` | Thin form | plan-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorPlans.tsx` | Thin list | plan-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorSettings.tsx` | Thin settings | supervisor-settings |
| `cpanel/src/app/ui/pages/supervisor/SupervisorSubscription.tsx` | Thin detail | subscription-management |
| `cpanel/src/app/ui/pages/supervisor/SupervisorSubscriptions.tsx` | Thin list | subscription-management |
| `cpanel/src/resources/configs/routes.ts` | Identifies | route-registry-contract |
| `cpanel/src/resources/configs/socket/events.ts` | `OnSupervisorEvent` | supervisor-settings |
| `cpanel/src/resources/configs/store/forms.ts` | Form keys | plan-management; supervisor-settings |
| `cpanel/src/resources/configs/supervisor/formRoute.ts` | Plan href | plan-management |
| `cpanel/src/resources/translations/ar.ts` | Copy | feature pages |
| `cpanel/src/types/events.ts` | Payload type | supervisor-settings |
| `cpanel/src/types/gql/definitions/supervisor.graphql` | Command-copy SDL | graphql-mirror (no line-by-line) |
| `cpanel/src/types/gql/gql-types/supervisor.ts` | Command-copy types | graphql-mirror (no line-by-line) |
| `cpanel/src/types/requesters/requesters.cpanel.ts` | Requester map mirror | http-and-requesters |

## 4) Still deferred (honest)

- Supervisor notifications UI
- Supervisor organizations directory (GQL roots exist)
- Customer writes from cpanel
- Drawer Support page (`SupervisorSupport`)
- Home meeting tile still links to the unfiltered meetings directory
