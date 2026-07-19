# Website Flow — Customer Meetings (Directory + Create + Empty Details)

Authenticated customer org-meeting directory, create-only form, and empty details stub on `CUSTOMER_MAIN`. Shell/breadcrumb contract: `flow-customer-shell.md` §7.1. Backend read/filter/create: `docs/platforms/backend/contracts/meeting-domain.md` (§4–§5 list filter, §9 write). Chairperson roster row: `meeting-participant-domain.md`. Form foundation: `flow-form-foundation.md` §3.5–§3.7.

## 1) Scope

**Shipped**

- Meeting directory for the authenticated customer's organization (GQL list + server `search` + single `status` filter + ResultLane load-more).
- Route-query persistence under history key `meetings` (`useWithHistoryState`); Enter commits search; status chips commit immediately.
- Clearing search or status (`All`) uses `{ reset: true }` so omitted keys are removed from the history query (avoids stale `status` after merge).
- Routes: list, create-only form (`/meetings/form` **before** `/meetings/:id`), empty details stub.
- List Add → form; **no Edit** on cards; card → details via typed `Link`.
- Status filter UI: shared `FilterOptionChips` / `FilterOptionChip` (text label + orange underline when active — landing filter language, not choice tiles).
- Create form: `Forms.CUSTOMER_MEETING` → `meeting.create`; fields `subject`, `type`, `datetime`, `min_members_count`, `chairperson`; success → details with `replace: true`.
- Drawer tile `CustomerMeetings` clickable (`CustomerDrawer`).
- Shared controls: `FormChoiceField`, `FormDateTimeField` + `DATETIME_PICKER`, `FormEntityPickerField` + `ENTITY_PICKER` (config registry + loadMore + `customScroll`).

**Not shipped**

- Non-empty meeting details UI (route + stub heading/subtitle only).
- GQL `canCreateMeeting` gating on Add (contract exists on `_Me`; Add always visible — same as members).
- Meeting update / delete requesters or UI.
- Plan `max_meetings_per_month` quota on create.
- Multi-status filter; `notify_status` filter; distinct empty copy for “no search hits” vs “no meetings yet”.
- Description field (product: title = `subject` only).

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| List route | `CustomerMeetings` → `/customer/meetings` |
| Form route | `CustomerMeetingForm` → `/customer/meetings/form` (create-only string path; **before** `:id`) |
| Details route | `CustomerMeetingDetails` → `/customer/meetings/:id` |
| List page | `website/src/app/ui/pages/customer/CustomerMeetings.tsx` |
| Form page | `website/src/app/ui/pages/customer/CustomerMeetingForm.tsx` |
| Details page | `website/src/app/ui/pages/customer/CustomerMeetingDetails.tsx` |
| List screen | `…/meetings/CustomerMeetingsScreen.tsx` |
| Form screen | `…/meetings/CustomerMeetingFormScreen.tsx` |
| Details screen | `…/meetings/CustomerMeetingDetailsScreen.tsx` |
| Hook | `…/hooks/useCustomerMeetings.ts` |
| Card | `…/meetings/CustomerMeetingCard.tsx` |
| Skeleton | `…/meetings/MeetingCardSkeleton.tsx` |
| Href builder | `website/src/resources/configs/customer/formRoute.ts` → `buildCustomerMeetingFormHref()` |
| Form identity | `Forms.CUSTOMER_MEETING` → `API.FORMS.CUSTOMER.R("meeting")` |
| Drawer | `CustomerDrawer` item `identify: "CustomerMeetings"`, `FiCalendar` |

All three: `layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`. Form/details breadcrumb parent `CustomerMeetings`; list breadcrumb parent `CustomerHome`.

Route order (W41): `CustomerMeetings` → `CustomerMeetingForm` → `CustomerMeetingDetails` in `routes.ts` / `MPagesRoutes`. Rule: `.cursor/rules/website-route-static-before-parametric.mdc`.

## 3) Directory data adapter

| Concern | Value |
|---|---|
| Mount-private adapter id | `"customer-meetings"` |
| Inherited | `DATA_ADAPTERS.CUSTOMER_GQL` → `API.DATA_ADAPTERS.CUSTOMER.GQL` |
| Listable | `"meetings"` |
| History key | `"meetings"` |
| Default page size | `adapter.maxLoadLength` or **24** |
| Reload | `useEffect` → `mLoad({ reload: true, query })` when `adapterQuery` changes (no `enterMode`; same as members) |
| Load-more | `mLoad({ query })` when `thereMoreRecords` and not busy |
| Refresh | `mLoad({ reload: true, query })` (ResultLane retry) |

History query shape: `{ search?: string; status?: string }`. Normalized: trimmed search; `status` only if it is a `_MeetingStatusValue` enum member (`DRAFT` | `WAITING_TO_START` | `STARTED` | `COMPLETED` | `CANCELED`); otherwise `""`.

Filter object sent to GQL (omit empty keys):

```ts
{
  ...(search ? { search } : {}),
  ...(status ? { status } : {})
}
```

Backend: `_MeetingFilter.search` → `subject` `iLike`; `_MeetingFilter.status` → equality. Root many only (`meeting(id)` ignores filter). See `meeting-domain.md` §4–§5.

GQL (inline in `useCustomerMeetings.ts`):

```graphql
query CustomerMeetings($filter: _MeetingFilter) {
    meetings(filter: $filter) {
        id
        subject
        type {
            value
            label
        }
        datetime
        min_members_count
        status {
            value
            label
        }
        chairperson {
            id
            name
            avatar_url
        }
        total_count
    }
}
```

History search rule: `.cursor/rules/website-customer-list-history-search.mdc`.

### 3.1 Status filter commit

- Chip row: `All` (`value: ""`) + one chip per `MEETING_STATUS_FILTER_VALUES`.
- Labels from page i18n (`statusAll`, `statusDraft`, …), not GQL labels (filter values are enum keys).
- `setStatusFilter(next)` / `submitSearch()` both call `setRouteQuery(..., { reset: true })` and **rebuild** the query from current search + status so clearing `All` drops `status` from the URL/history.

## 4) Directory screen composition

`Container` → `Col pt={2} gap={1.5} pb={2}`:

1. Row `jc_sb`: `SectionHeading` (title/subtitle) + primary `FormActionButton` Add → `nav.push(buildCustomerMeetingFormHref())`. **Not** gated on `me.canCreateMeeting`.
2. `SearchField` (draft + Enter → `submitSearch`).
3. `FilterOptionChips` (status).
4. `ResultLane` → `renderSkeleton: MeetingCardSkeleton`; `renderCard: CustomerMeetingCard`.

Skeleton shape rule: `.cursor/rules/website-result-lane-skeleton-shape.mdc`. `ResultLane` accepts optional `renderSkeleton` (shared lane change in this slice).

### 4.1 Card contract (W42)

Presentational `CustomerMeetingCard` does **not** call `useTranslator`. Screen passes:

| Prop | Source |
|---|---|
| `subject`, `datetime`, `chairpersonName`, `chairpersonAvatarUrl`, `id` | GQL row |
| `statusLabel` | `status.label` \|\| `status.value` \|\| `"—"` |
| `typeLabel` | `type.label` \|\| `type.value` |
| `chairpersonLabel` | `t("cardChairperson")` |
| `quorumLabel` | `t("cardQuorum", { count })` when `min_members_count > 0` |

Card UI: accent start rail; subject; meta line `type · status`; `useMoment` date + time rows; footer divider with chairperson avatar/name + quorum caption. Navigation: `Col As={Link}` typed `To<"CustomerMeetingDetails">`. Rule: `.cursor/rules/website-presentational-label-props.mdc`.

## 5) Create form — `Forms.CUSTOMER_MEETING`

| Concern | Value |
|---|---|
| Form identity | `Forms.CUSTOMER_MEETING` → `api: API.FORMS.CUSTOMER.R("meeting")` |
| Path | Create-only (no multi-path update) |
| Reducer key | Stable `"customer-meeting-form-create"` + `removeOnExit: false` |
| Init | `values: {}` |
| Loading | `isLoading = !exist \|\| formLoading` (`SENDING`) |
| Submit | `sub: "create"`; `submittingRef` guard |

Governance: `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`, `.cursor/rules/website-form-success-toast-automatic.mdc` (no manual success toast).

### 5.1 Screen chrome

- Header row: `SectionHeading` (`onBack` → `nav.back()`, `backLabel` from `ui.components.mainHeader.back`) + primary Create button (not full-width under fields).
- Fields column `maxW={32}`:

| Field `name` | Control | Notes |
|---|---|---|
| `subject` | `FormTextField` | Required by backend (trim min 2) |
| `type` | `FormChoiceField` | Options `PERIODIC` / `EMERGENCY` from i18n |
| `datetime` | `FormDateTimeField` | ISO string; future-only client gate |
| `min_members_count` | `FormTextField` `type="number"` | Backend integer ≥ 1 |
| `chairperson` | `FormEntityPickerField` `ident="members"` | `{ value, label, avatarUrl? }` |

### 5.2 Success navigation

`afterSentSuccess`: read `res.data.other.meetingId` → `nav.push({ identify: "CustomerMeetingDetails", params: { id }, replace: true })`.

- `replace: true` drops the form from history so back from details returns to the **list**.
- If `meetingId` missing: stay on form (axios success toast may still show).

### 5.3 Backend create contract (summary)

Validate → `can MEETING create` → `organization.createMeeting` (`status=DRAFT`, `notify_status=NOT_STARTED`) → `createParticipant({ type: "CHAIRPERSON" })` → `other.meetingId` + `SUCCESS_CREATE`. Full table: `meeting-domain.md` §9.

Client past-date check is **stricter** than backend `joi.date()` (any date accepted server-side).

## 6) Empty details stub

`CustomerMeetingDetailsScreen`: `SectionHeading` title/subtitle from `ui.pages.customer.meetingDetails.*` only. No GQL load of the meeting by id in this slice. Route exists so create redirect and list card links resolve.

## 7) Shared field + modal contracts (this slice)

Deep contracts live in `flow-form-foundation.md` §3.5–§3.7. Meeting-specific usage:

| Surface | Meeting usage |
|---|---|
| `FormChoiceField` | Meeting `type` tiles (not status filter chips) |
| `FormDateTimeField` / `DATETIME_PICKER` | Meeting `datetime`; 15-min slots; full-width `.ejt-datepicker` via `datepickerTheme` |
| `FormEntityPickerField` / `ENTITY_PICKER` | Chairperson; config `members.tsx`; `CustomerMemberCard` + `selected`; `customScroll` + `LoadMoreButton` |
| `FormTextField` | `type="number"` allowed for quorum |
| `FilterOptionChip(s)` | List status filter only |
| `CustomerMemberCard` | Optional `selected` soft fill for picker highlight |

Third-party calendar: library CSS in `dependencies.scss`; theme in `resources/emotion/styles/datepicker.ts`. Rule: `.cursor/rules/website-third-party-widget-emotion-theme.mdc`.

Modal scroll bodies: `minH={0}` + `customScroll`. Rule: `.cursor/rules/website-custom-scroll-contract.mdc`.

## 8) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.meetings.*` | Directory title, search, status chips, empty, load-more, card chairperson/quorum labels |
| `ui.pages.customer.meetingForm.*` | Create title, field labels, type options, datetime/chairperson chrome, create button |
| `ui.pages.customer.meetingDetails.*` | Stub title/subtitle |
| `ui.modals.entityPicker.*` | Confirm/cancel/search/empty/loadMore |
| `ui.modals.dateTimePicker.*` | Confirm/cancel/datePart/timePart/pickDate/pickTime |
| `ui.components.mainHeader.back` | Form SectionHeading back |

ar/en mirrors required. Note: `meetingForm.typeEmpty` exists in translations but is unused by `FormChoiceField` (no empty placeholder tile).

## 9) Failure / empty modes (UI)

| Condition | UI |
|---|---|
| Directory initial load | `MeetingCardSkeleton` grid via `ResultLane` |
| Directory empty | `Empty` overlay |
| Directory fail | `LaneFailed` + retry (`refresh`) |
| Picker empty members | Modal `Empty` |
| Picker loading | Absolute `Loadable` overlay on list |
| Picker more pages | `LoadMoreButton` |
| Form validation | Field errors via form reducer (Joi mirror) |
| Create ability fail (no org) | Backend main message toast; form stays |
| Create success without `meetingId` | Toast may fire; form stays |

## 10) Dependencies (website)

| Package | Role |
|---|---|
| `react-datepicker` | Inline calendar inside `DateTimePickerModal` |
| Removed | `flatpickr`, `react-flatpickr`, `@types/react-flatpickr` (unused after this slice) |

## 11) Verify

- `yarn type-check` in `website/` and `backend/` (when SDL/requester changed).
- Smoke: drawer → list; Enter search; status chip + All clear; create → details (`replace`); back details → list; back form → list; chairperson picker empty/load-more/avatar after pick; datetime full-width calendar + time scroll.

## 12) Traceability map (this change set)

### Backend (`backend/` repo)

| Path | Role | Doc |
|---|---|---|
| `src/app/orchestrator/requesters/MeetingRequester.ts` | `create` | `meeting-domain.md` §9 |
| `src/app/orm/models/Customer.ts` | `Ability.MEETING` | §9.1 |
| `src/app/validation/joi_rules.ts` | `isCustomerOwnedMember(..., "chairperson")` | §9.2 |
| `requesters.website.ts` | `customer.meeting: "create"` | §9.4 |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | `_MeetingFilter` on many | §4–§5 |
| `src/app/gql/bridges/customer/MeBridge.ts` | `canCreateMeeting` | §9.1 |
| `src/app/gql/definitions/customer.graphql` | SDL | §4 |
| `src/app/gql/schemas/CustomerSchema.ts` | resolvers | §4 |
| `src/app/gql/gql-types/customer.ts` | Generated | skip line narrative |
| `src/app/orchestrator/requesters/MemberRequester.ts` | Org-missing throw tweak (intentional; not meetings-owned) | `member-domain.md` |

### Website (`website/` repo)

| Path | Role | Doc |
|---|---|---|
| `src/types/requesters/requesters.website.ts` | `customer.meeting` | §5 |
| `src/resources/configs/store/forms.ts` | `CUSTOMER_MEETING` | §5 |
| `src/resources/configs/store/modals.ts` | `ENTITY_PICKER`, `DATETIME_PICKER` | §7 |
| `src/resources/configs/customer/formRoute.ts` | `buildCustomerMeetingFormHref` | §2 |
| `src/resources/configs/routes.ts` | three meeting routes | §2 |
| `src/app/ui/pages/customer/CustomerMeetings.tsx` | thin page | §2 |
| `src/app/ui/pages/customer/CustomerMeetingForm.tsx` | thin page | §2 |
| `src/app/ui/pages/customer/CustomerMeetingDetails.tsx` | thin page | §2, §6 |
| `src/app/ui/components/customer/meetings/*` | screens, card, skeleton | §4–§6 |
| `src/app/ui/components/customer/hooks/useCustomerMeetings.ts` | list hook | §3 |
| `src/app/ui/components/FilterOptionChip.tsx` | status chip | §3.1, §4 |
| `src/app/ui/components/FilterOptionChips.tsx` | chip row | §3.1, §4 |
| `src/app/ui/components/ResultLane.tsx` | `renderSkeleton` | §4 |
| `src/app/ui/components/form/FormChoiceField.tsx` | type tiles | §5.1; form-foundation §3.5 |
| `src/app/ui/components/form/FormDateTimeField.tsx` | datetime field | §5.1; §3.7 |
| `src/app/ui/components/form/FormEntityPickerField.tsx` | chairperson | §5.1; §3.6 |
| `src/app/ui/components/form/FormTextField.tsx` | `number` type | §5.1 |
| `src/app/ui/components/modals/EntityPickerModal.tsx` | picker shell | §7 |
| `src/app/ui/components/modals/DateTimePickerModal.tsx` | datetime shell | §7 |
| `src/app/ui/components/modals/SelectableEntityCard.tsx` | selection chrome | §7 |
| `src/app/ui/components/modals/entity-picker/configs/members.tsx` | members ident | §7 |
| `src/app/ui/components/modals/entity-picker/configs/index.ts` | registry | §7 |
| `src/app/ui/components/modals/entity-picker/types.ts` | selection + config types | §7 |
| `src/app/ui/components/customer/members/CustomerMemberCard.tsx` | `selected` prop | §7 |
| `src/resources/emotion/styles/datepicker.ts` | calendar theme | §7 |
| `src/resources/emotion/styles/scroll.ts` | `customScroll` | §7 |
| `src/resources/styles/dependencies.scss` | `react-datepicker` CSS import | §7, §10 |
| `src/resources/translations/ar.ts` / `en.ts` | §8 keys | §8 |
| `src/types/gql/definitions/customer.graphql` | SDL mirror | backend §4 |
| `src/types/gql/gql-types/customer.ts` | Generated mirror | skip line narrative |
| `package.json` / `yarn.lock` | `react-datepicker`; flatpickr removed | §10 |
| `lib/tsconfig.tsbuildinfo` | Generated | skip narrative |
| Deleted `src/resources/styles/react-datepicker-theme.scss` | Replaced by `datepicker.ts` | §7 |

### Root docs / governance

| Path | Role |
|---|---|
| `docs/platforms/website/flow-customer-meetings.md` | this flow |
| `docs/platforms/website/flow-form-foundation.md` | shared fields/modals |
| `docs/platforms/website/route-registry-contract.md` | routes §5.2 |
| `docs/platforms/website/component-structure.md` | ownership |
| `docs/platforms/website/overview.md` / `README.md` | indexes |
| `docs/platforms/website/data-flow-and-gql.md` | adapter index |
| `docs/platforms/website/graphql-mirror-and-tooling.md` | mirror index |
| `docs/platforms/backend/contracts/meeting-domain.md` | backend contract |
| `docs/platforms/backend/contracts/meeting-participant-domain.md` | chairperson roster |
| `.cursor/rules/website-custom-scroll-contract.mdc` | modal + drawer scroll |
| `.cursor/rules/website-presentational-label-props.mdc` | W42 |
| `.cursor/rules/website-third-party-widget-emotion-theme.mdc` | datepicker theme |
| `.cursor/skills/website-customer-meeting-form/SKILL.md` | repeatable meeting UI |
| `.cursor/skills/website-entity-picker/SKILL.md` | repeatable picker |

## 13) Related

- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/website/flow-customer-members.md` (list/form pattern sibling)
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `.cursor/rules/website-route-static-before-parametric.mdc`
- `.cursor/rules/website-customer-list-history-search.mdc`
- `.cursor/rules/website-result-lane-skeleton-shape.mdc`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
