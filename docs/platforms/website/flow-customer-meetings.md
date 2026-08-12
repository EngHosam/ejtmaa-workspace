# Website Flow — Customer Meetings (Directory + Create + Details Roadmap)

Authenticated customer org-meeting directory, create form, and details readiness roadmap on `CUSTOMER_MAIN`. Shell/breadcrumb contract: `flow-customer-shell.md` §7.1. Backend: `docs/platforms/backend/contracts/meeting-domain.md` (§4–§5 list filter, §9 write/approve). Chairperson roster: `meeting-participant-domain.md`. Form foundation: `flow-form-foundation.md` (§3.5–§3.8 + FORM modal).

## 1) Scope

**Shipped**

- Meeting directory for the authenticated customer's organization (GQL list + server `search` + single `status` filter + ResultLane load-more).
- Route-query persistence under history key `meetings` (`useWithHistoryState`); Enter commits search; status chips commit immediately.
- Clearing search or status (`All`) uses `{ reset: true }` so omitted keys are removed from the history query (avoids stale `status` after merge).
- Routes: list, create-only form (`/meetings/form` **before** `/meetings/:id`), details roadmap.
- List Add → form; **no Edit** on cards; card → details via typed `Link`.
- Status filter UI: shared `FilterOptionChips` / `FilterOptionChip` (text label + orange underline when active — landing filter language, not choice tiles).
- Create form: `Forms.CUSTOMER_MEETING` → `meeting.create`; fields `subject`, `type`, `datetime`, `min_members_count`, `chairperson`; success → details with `replace: true`.
- Details readiness roadmap (§6): private modal forms + page form for approve/delete/remove/template FK; Ability-gated edit lock.
- Drawer tile `CustomerMeetings` clickable (`CustomerDrawer`).
- Shared controls: `FormChoiceField`, `FormDateTimeField` + `DATETIME_PICKER`, `FormEntityPickerField` + `ENTITY_PICKER` (config registry + loadMore + `customScroll`).

**Not shipped**

- GQL `canCreateMeeting` gating on Add (contract exists on `_Me`; Add always visible — same as members).
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
        notify_status {
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

`notify_status { value label }` is selected on the **list** query (not only details) so directory cards can render the invite-notify chip. History search rule: `.cursor/rules/website-customer-list-history-search.mdc`.

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
| `statusValue` | `status.value` (chip icon/tone discriminator) |
| `typeLabel` | `type.label` \|\| `type.value` |
| `typeValue` | `type.value` (chip icon/tone discriminator) |
| `notifyStatusLabel` | `notify_status.label` \|\| `notify_status.value` |
| `notifyStatusValue` | `notify_status.value` (chip icon/tone discriminator) |
| `chairpersonLabel` | `t("cardChairperson")` |
| `quorumLabel` | `t("cardQuorum", { count })` when `min_members_count > 0` |

Card UI: accent start rail; subject; **meta chips row** (`MeetingMetaChips` — type · status · notify as separate icon+label pills, **not** a joined `·` text line); `useMoment` date + time rows; footer divider with chairperson avatar/name + quorum caption. Navigation: `Col As={Link}` typed `To<"CustomerMeetingDetails">`.

`*Value` props are enum **discriminators** for icon + semantic color only — labels stay resolved strings, so W42 (no translator in presentational components) still holds. Chip component + tone/icon/color contract: `MeetingMetaChips` (§4.2). Rules: `.cursor/rules/website-presentational-label-props.mdc`, `.cursor/rules/website-meeting-meta-chips.mdc`.

### 4.2 Meeting meta chips (`MeetingMetaChips`)

Shared presentational component for the three meeting axes, used by `CustomerMeetingCard` (directory + home + details basics reuse) and the `CustomerMeetingDetails` header. Renders one pill per non-empty axis: leading Feather icon + label, pill background + icon + text from a scheme-aware tone.

**Distinct icon per value (no repeat across axes):**

| Axis | Value | Icon | Tone |
|---|---|---|---|
| type | `PERIODIC` | `FiRepeat` | info |
| type | `EMERGENCY` | `FiAlertTriangle` | danger |
| status | `DRAFT` | `FiEdit3` | neutral |
| status | `WAITING_TO_START` | `FiClock` | warning |
| status | `STARTED` | `FiPlayCircle` | success |
| status | `COMPLETED` | `FiCheckCircle` | info |
| status | `CANCELED` | `FiXCircle` | danger |
| notify | `NOT_STARTED` | `FiBellOff` | neutral |
| notify | `WAITING_TO_NOTIFY` | `FiSend` | warning |
| notify | `NOTIFIED` | `FiBell` | success |

**Tone → tokens (light / dark auto via `ThemeMap`):**

| Tone | bg | icon + text |
|---|---|---|
| neutral | `sectionBrandBackground` | `iconSecondary` / `textSecondary` |
| info | `stateSoftInfo` | `stateInfo` |
| warning | `stateSoftWarning` | `stateWarning` |
| success | `stateSoftSuccess` | `stateSuccess` |
| danger | `stateSoftError` | `stateError` |

Soft-tint tokens (`stateSoft*`) resolve to pastel fills in light and translucent dark fills in dark; `state*` foregrounds resolve to the default feedback color in light and the brighter `onDark` variant in dark — correct contrast in both schemes without per-scheme branches (added in `semanticColor`, §semantic color). Icon size `0.78`; unified `caption` weight (no bold). Empty axis label → chip omitted; all empty → renders `null`.

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
| `datetime` | `FormDateTimeField` | ISO string; `minDate={meetingDatetimeMinDate()}` + `minDateError` (10-minute lead mirror, §6.5) |
| `notify_start_at` | `FormDateTimeField` | ISO string; required; no `minDate` (the only lower bound is "future", covered by `pastDateError`). The 5-minute gap before `datetime` is server-enforced and stated in `subTitle`; a violation returns the localized Joi `notify_start_at.tooLate` on the field |
| `min_members_count` | `FormTextField` `type="number"` | Backend integer ≥ 1 |
| `chairperson` | `FormEntityPickerField` `ident="members"` | `{ value, label, avatarUrl? }` |

### 5.2 Success navigation

`afterSentSuccess`: `d.reset()` then read `res.data.other.meetingId` → `nav.push({ identify: "CustomerMeetingDetails", params: { id }, replace: true })`.

- `replace: true` drops the form from history so back from details returns to the **list**.
- If `meetingId` missing: stay on form (axios success toast may still show).

### 5.3 Backend create contract (summary)

Validate (`isMeetingDatetime` — 10-minute lead; `isMeetingNotifyStartAt` — future + 5-minute gap) → `can MEETING create` → `organization.createMeeting` (`status=DRAFT`, `notify_status=NOT_STARTED`, client-chosen `notify_start_at`) → `createParticipant({ type: "CHAIRPERSON" })` → `other.meetingId` + `SUCCESS_CREATE`. Full table: `meeting-domain.md` §9.

Client and server enforce the **same** lead rule: the field blocks the pick with `datetimeMinLeadError`, the server rejects with the Joi `datetime.tooSoon` field error. The client gate is a mirror, never the only check (§6.5).

## 6) Details roadmap — `CustomerMeetingDetails`

Preparation journey for `DRAFT` → `WAITING_TO_START` (not a live session UI).

**Home cross-link:** `CustomerHome` may surface status slices (`DRAFT` / `WAITING_TO_START` / `STARTED`), a focus meeting readiness preview, and notify-status captions — see `flow-customer-shell.md` §7. Prep writes and **approve** remain on this details screen.

### 6.1 Data

Hook `useCustomerMeetingDetails` — mount-private adapter `"customer-meeting-details"` inherit `CUSTOMER_GQL`, query `meeting(id)` **without** `listable` (section.meeting). Selection includes nested participants/agenda/decisions/templates + `canUpdate` / `canDelete` / `canApprove` / `canCancel`.

### 6.2 Chrome

`SectionHeading` back + subject title, wrapped in a `Col` with a `MeetingMetaChips` row beneath it (type · status · notify pills — replaces the former status·type subtitle text; §4.2); Approve / Cancel / Delete via `FormActionButton` when Ability allows; Cancel is gated on `canCancel` and asks through `confirm()` before submitting the `cancel` sub. Approve uses `canApprove.value` only to enable/disable — do **not** render `canApprove.description` under the button (denial copy stays server-side; readiness strip covers UX guidance). Lock and schedule notes both render through **`MeetingNote`** — the single alert-chrome component for this folder (`canvasAccentSoftBackground` + `FiAlertTriangle` + `semanticDims.card.compactPadding`, `role="status"`); `title` is optional and bullets appear only for 2+ items, so a one-line lock banner and a multi-line schedule list share one component (§6.5). `CustomerMeetingTemplateSlots` renders its contact-mode callout through the same component — do not hand-roll a fourth copy of that chrome. Basics body reuses `CustomerMeetingCard` language (accent rail, calendar/clock, chair + quorum) and adds a `FiSend` + `notify_start_at` line in the same single-line icon+text shape as the date and time rows (no stacked label/value card). Section Add/Edit are `smallAction` text actions (member-card pattern), not `FormActionButton`. Roster rows follow `CustomerMemberCard` and are grouped by participant type through **`MeetingParticipantGroup`** (heading + hairline divider) in the order chairperson → viewers → members; the group heading carries the type, so `CustomerMeetingParticipantRow.typeLabel` is optional and the per-row type caption is omitted inside a group; agenda/decision rows are quiet cards with order caption + optional status meta. Roadmap sections after basics use `MeetingDetailsSection` `divided` (`divider` hairline — primary, not `subtleDivider`) so agenda / decision phases / templates read as separate blocks in light mode.

### 6.3 Sections

Basics / participants / agenda / decisions write UI opens **registered meeting modals** whose bodies are private `useShallowForm` + `Form*` stacks (`MeetingBasicsModal` `read`→`update`, `MeetingParticipantAddModal` `addParticipant`, `MeetingSubjectModal` agenda/decision create|update). Each modal composes presentational `FormModal` chrome and opens via its own helper (`openMeetingBasics`, `openMeetingParticipantAdd`, `openMeetingSubject`) — never a shared `render` factory. Page-level form stays for approve/cancel/delete/remove only — never share it into modal bodies. Templates follow the same modal pattern: the section Edit action opens `MeetingTemplatesModal` (`read` → `updateTemplates`) whose body holds two `FormEntityPickerField`s on `messageTemplates` with the mirrored `types` family filter from `meetingNotifyTemplateMode.ts` and `clearLabel={tPicker("clear")}` (the field renders the clear action in `FormInputWrapper.actionArea`); `CustomerMeetingTemplateSlots` is display-only. Because `updateTemplates` carries the two template refs and nothing else, a template change never re-validates `datetime` (`meeting-domain.md` §9.3).

**Decisions (prepare):** two sections — pre-start (`PRE_START`) and in-meeting (`DURING`). Each Add opens `createDecision` with that `phase` required in form values. While `canUpdate` (notify still `NOT_STARTED`), both phases share the same edit/delete chrome; update does not change `phase`. Approve readiness still counts **pre-start only** (≥1). Live-session writes are out of this flow.

Template mode callout: `meetingNotifyTemplateMode.ts` (mirror of backend helper). Enforcement remains Ability/`approve`.

### 6.4 Modals

Customer-only form modals live under `src/app/ui/components/customer/modals/` (not shared `components/modals/`, not under `meetings/`). Shared registry: `resources/configs/store/modals.ts`. Deep chrome + placement contract: `flow-form-foundation.md` §3.8b.

| Identity | Component / open helper | Open props (data + callbacks) | Form `sub` |
|---|---|---|---|
| `MEETING_BASICS` | `MeetingBasicsModal` / `openMeetingBasics` | `title`, `meetingId`, optional `subtitle`, `onSuccess` | enter `read` → save `update` |
| `MEETING_PARTICIPANT_ADD` | `MeetingParticipantAddModal` / `openMeetingParticipantAdd` | `title`, `meetingId`, optional `subtitle`, `onSuccess` | `addParticipant` (default type `MEMBER`) |
| `MEETING_SUBJECT` | `MeetingSubjectModal` / `openMeetingSubject` | `title`, `meetingId`, `mode`, `subjectLabel`, `submitLabel`, optional `initialSubject`, `onSuccess`; `entityId` / `phase` by mode | `createAgendaItem` / `updateAgendaItem` / `createDecision` / `updateDecision` |
| `MEETING_TEMPLATES` | `MeetingTemplatesModal` / `openMeetingTemplates` | `title`, `meetingId`, optional `subtitle`, `onSuccess` | enter `read` → save `updateTemplates` |

Also on this screen:

- `FormModal` — presentational chrome only (shared `components/modals/`; **not** registered); composed by the meeting modals above
- `CONFIRM` — approve warning, deletes
- `ENTITY_PICKER` — `members`, `messageTemplates` (`selected` on cards)
- `DATETIME_PICKER` — nested from basics datetime field

**Invariant:** one registered Component per identity; never a generic `FORM` + Redux `render` JSX factory. Do not edit `ModalBase` for product form bugs. After success: `onSuccess` (details `refresh`) then `closeMe({})`. Forms: `Forms.CUSTOMER_MEETING` with truthful `sub`; init identity `{ meeting: id }` — no id echo from `read`.

Deleted anti-pattern files (do not resurrect): `meetings/MeetingBasicsModalForm.tsx`, `MeetingParticipantAddModalForm.tsx`, `MeetingSubjectModalForm.tsx`.

### 6.5 Schedule policy mirror + schedule notes

Backend owns the schedule rules (`meeting-domain.md` §9.1a). The website mirrors the two constants in **one** module and uses them for copy and for the picker floor — never as the only enforcement.

**Module:** `…/customer/meetings/meetingScheduleLead.ts`

The second mirror module in the same folder, `meetingNotifyTemplateMode.ts`, owns the notify-template family lists (`MEETING_WHATSAPP_TEMPLATE_TYPES`, `MEETING_EMAIL_TEMPLATE_TYPES`, mirroring the requester constants) next to the contact-mode resolver. A screen or modal that needs a template-family filter imports from there; an inline `["ADWHATS", …]` array in a picker call is the drift this module exists to prevent.

The 5-minute invite gap has **no** client constant: it appears only as copy (`notifyStartAtSubtitle`, `scheduleNoteInviteRule`), the same way the 10-minute lead appears in `datetimeMinLeadError`. A mirrored constant with no consumer would be dead code — add one only when a control actually computes with it.

| Export | Value / behavior | Mirrors |
|---|---|---|
| `MEETING_MIN_LEAD_MS` | `10 * 60 * 1000` | `MeetingModel.MIN_LEAD_MS` |
| `meetingDatetimeMinDate()` | `new Date(Date.now() + MEETING_MIN_LEAD_MS)` | `minDate` for both datetime fields (create screen + `MeetingBasicsModal`) |
| `isVotingParticipantType(value)` | `CHAIRPERSON` \| `MEMBER` | Ability quorum filter |
| `countVotingParticipants(rows)` | count of voting rows | `countParticipants({ where: { type: IN (…) } })` |

`countVotingParticipants` is the only quorum counter on the website: details roster readiness and `useCustomerHome` (`FocusReadiness.votingCount`) both call it, so `VIEWER` rows never inflate a quorum ring or a readiness row.

**Schedule notes (`MeetingNote` on the details screen).** Derived state: `hasValidDatetime`, `datetimeTooSoon` (`datetime < now + 10 min`), `hasValidNotifyStart` (`notify_start_at` parses).

| Condition | Items | Title |
|---|---|---|
| `isLocked` (notify left `NOT_STARTED`) | lock banner only, in its own `MeetingNote` | — |
| status is not `DRAFT` / `WAITING_TO_START` | none | — |
| `DRAFT` | lead line — missing / too-soon / rule | `scheduleNotesTitleDraft` |
| always (both statuses) | invite-start line: `scheduleNoteInviteAt` with the formatted `notify_start_at`, else `scheduleNoteInviteRule` | `scheduleNotesTitleWaiting` for `WAITING_TO_START` |
| always (both statuses) | edit line: `scheduleNoteEditUntilApproval` on `DRAFT`, `scheduleNoteEditClosed` otherwise | — |

**Readiness card.** The approve checklist (basics / quorum / agenda / pre-start decisions / templates) renders only while the meeting is `DRAFT` or `WAITING_TO_START`, the same window the schedule notes use. A `STARTED`, `COMPLETED`, or `CANCELED` meeting shows no approval checklist, because no approve gate remains for it to describe.

Three alignments with the server that must not drift:

- The lead lines are **draft-only**. An approved meeting legitimately drops under 10 minutes as it approaches; showing "edit the time before approving" there would name a rule that no longer applies.
- The edit line states that approval closes editing for good, matching the `DRAFT`-only `update` Ability (`meeting-domain.md` §9.1a). The escape hatch is the cancel action, gated on `canCancel` and confirmed through `confirm()`.
- The cancel confirm says the action cannot be undone and nothing more. It must not promise that attendees lose access, because a draft that never sent invites has no attendees to lose.

## 7) Shared field + modal contracts (this slice)

Deep contracts live in `flow-form-foundation.md` §3.5–§3.7. Meeting-specific usage:

| Surface | Meeting usage |
|---|---|
| `FormChoiceField` | Meeting `type` tiles (not status filter chips) |
| `FormDateTimeField` / `DATETIME_PICKER` | Meeting `datetime`; 5-min slots; full-width `.ejt-datepicker` via `datepickerTheme`; `minDate` = `meetingDatetimeMinDate()` and `minDateError` = `datetimeMinLeadError` on both write surfaces (create screen, `MeetingBasicsModal`) |
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
| `ui.pages.customer.meetingDetails.*` | Details roadmap copy (sections, readiness, templates, confirms) |
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
| Details modal open with missing Component/props | Must not happen — registered Component + typed open helpers; former `render is not a function` white screen was the generic `FORM` anti-pattern |
| Details modal cancel / backdrop | `cancelable: true` → close; no save |
| Details modal save while sending | Save/Add `loading` scoped by `currentSub`; `submittingRef` blocks double-send |

## 10) Dependencies (website)

| Package | Role |
|---|---|
| `react-datepicker` | Inline calendar inside `DateTimePickerModal` |
| Removed | `flatpickr`, `react-flatpickr`, `@types/react-flatpickr` (unused after this slice) |

### 10.1 Success feedback asset

`Toast` chooses the success Lottie from the active color scheme:

- `dark-success.json` is the existing colored success animation for light scheme.
- `light-success.json` is the white success animation for dark scheme.

The asset choice is presentation-only; toast status, message delivery, and
accessibility semantics remain owned by the shared toast surface.

## 11) Verify

- `yarn type-check` in `website/` and `backend/` (when SDL/requester changed).
- Smoke: drawer → list; Enter search; status chip + All clear; create → details (`replace`); back details → list; back form → list; chairperson picker empty/load-more/avatar after pick; datetime full-width calendar + time scroll; details Add/Edit opens registered modals (basics/participant/subject) without white screen; cancel closes; save refreshes details.

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
| `docs/platforms/website/component-structure.md` | ownership (incl. `customer/modals`) |
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

### 12.1 Details implementation inventory

The preceding map covers the original directory/form foundation. This table is the
complete source inventory for the details, notify-template, decision-phase, and
feedback additions in the current slice. Backend source is detailed in
[`meeting-domain.md` §10](../backend/contracts/meeting-domain.md#10-traceability-map)
and the relevant child contracts.

| Path | Role | Section |
|---|---|---|
| `src/app/ui/components/customer/hooks/useCustomerMeetingDetails.ts` | Root-one Meeting GQL adapter and refresh boundary. | §6.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | Details composition, readiness, Ability-gated writes, chairperson-first roster, and divided sections. | §6.2–§6.3 |
| `src/app/ui/components/customer/meetings/CustomerMeetingParticipantRow.tsx` | Roster presentation and chairperson-safe remove action. | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingAgendaRow.tsx` | Ordered agenda presentation and actions. | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDecisionRow.tsx` | Decision row presentation contract. | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingTemplateSlots.tsx` | Display-only template slots and contact-mode callout. | §6.3 |
| `src/app/ui/components/customer/modals/MeetingBasicsModal.tsx` | Registered basics modal (`read`→`update`) + `openMeetingBasics`. | §6.3–§6.4 |
| `src/app/ui/components/customer/modals/MeetingParticipantAddModal.tsx` | Registered participant add modal + `openMeetingParticipantAdd`. | §6.3–§6.4 |
| `src/app/ui/components/customer/modals/MeetingSubjectModal.tsx` | Registered agenda/decision subject modal + `openMeetingSubject`. | §6.3 |
| `src/app/ui/components/customer/modals/MeetingTemplatesModal.tsx` | Registered templates modal (`read`→`updateTemplates`) + `openMeetingTemplates`. | §6.3–§6.4 |
| `src/app/ui/components/customer/meetings/meetingNotifyTemplateMode.ts` | UI-only mirror: contact-mode resolver for readiness copy **and** the two template-family lists used by the picker filters; backend remains enforcement source. | §6.3, §6.5 |
| `src/app/ui/components/form/FormEntityPickerField.tsx` | Shared ref field; `clearLabel` renders the clear action for nullable refs. | `flow-form-foundation.md` §3.6 |
| `src/app/ui/components/modals/FormModal.tsx` | Presentational form-modal chrome (not registered). | §6.3–§6.4 |
| `src/resources/configs/store/modals.ts` | Meeting modal identities + shared pickers/confirm. | §6.4 |
| `src/app/ui/components/modals/entity-picker/configs/index.ts` | `messageTemplates` picker registration. | §6.3 |
| `src/app/ui/components/modals/entity-picker/configs/messageTemplates.tsx` | Non-searching template picker and type-family filter mapping. | §6.3 |
| `src/app/ui/components/customer/message-templates/CustomerMessageTemplateCard.tsx` | Selected card presentation in the picker. | §6.3 |
| `src/types/requesters/requesters.website.ts` | Exact customer `meeting` sub-map mirror. | §6.3 |
| `src/types/gql/definitions/customer.graphql` | Backend customer SDL mirror. | §6.1 |
| `src/types/gql/gql-types/customer.ts` | Generated customer type mirror; not independently authored. | §6.1 |
| `src/resources/translations/ar.ts` / `en.ts` | Details, template-mode, message-type, and status copy. | §8 |
| `src/app/ui/components/Toast.tsx` | Color-scheme selection for the success Lottie. | §10 |
| `src/resources/animations/dark-success.json` | Existing success asset renamed for light-scheme use. | §10 |
| `src/resources/animations/light-success.json` | White success asset for dark-scheme use. | §10 |
| `src/resources/animations/success.json` | Renamed to `dark-success.json`; no remaining consumer. | §10 |
| `lib/tsconfig.tsbuildinfo` | Generated TypeScript build state; excluded from source narrative and commits. | generated |
| `docs/platforms/website/flow-customer-meetings.md` | This observable flow and inventory. | all |
| `docs/platforms/website/flow-form-foundation.md` | Form-modal chrome + registered meeting modals foundation. | §6.3–§6.4 |
| `.cursor/rules/meeting-lifecycle-approve-lock.mdc` | Durable lifecycle, approve, and notify-lock guardrail. | §6.3 |
| `.cursor/rules/decision-meeting-child.mdc` | DURING/PRE_START prepare-write and approve-completeness guardrail. | §6.3 |
| `.cursor/rules/website-customer-form-modal-placement.mdc` | Customer form-modal placement + no Redux JSX `render`. | §6.4; form-foundation §3.8b |
| `.cursor/skills/website-customer-form-modal/SKILL.md` | Repeatable customer registered form-modal workflow. | §6.4 |
| `.cursor/skills/website-customer-meeting-form/SKILL.md` | Create/list + pointer to details modals skill. | §5–§6 |

### 12.2 Registered meeting form-modals refactor (this go-doc slice)

Exhaustive inventory for replacing generic `FORM` + `openForm({ render })` with ConfirmModal-style registered Components under `customer/modals/`.

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/modals/MeetingBasicsModal.tsx` | added | Registered basics modal + `openMeetingBasics` | §6.3–§6.4; form-foundation §3.8b |
| `src/app/ui/components/customer/modals/MeetingParticipantAddModal.tsx` | added | Registered participant add + `openMeetingParticipantAdd` | §6.3–§6.4; §3.8b |
| `src/app/ui/components/customer/modals/MeetingSubjectModal.tsx` | added | Registered subject modal + `openMeetingSubject` | §6.3–§6.4; §3.8b |
| `src/app/ui/components/modals/FormModal.tsx` | modified | Presentational chrome only (`FormModalChromeProps`); not registered | §6.4; §3.8b |
| `src/resources/configs/store/modals.ts` | modified | Dropped generic `FORM`; added three meeting identities | §6.4 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | modified | Wires open helpers; no `openForm` / `*ModalForm` | §6.2–§6.4 |
| `src/app/ui/components/customer/meetings/MeetingBasicsModalForm.tsx` | deleted | Former body used with `openForm({ render })` | do not resurrect |
| `src/app/ui/components/customer/meetings/MeetingParticipantAddModalForm.tsx` | deleted | Former body used with `openForm({ render })` | do not resurrect |
| `src/app/ui/components/customer/meetings/MeetingSubjectModalForm.tsx` | deleted | Former body used with `openForm({ render })` | do not resurrect |
| `src/app/ui/base/components/ModalBase.tsx` | unchanged | Intentionally not edited for this bug | out of scope |
| `lib/tsconfig.tsbuildinfo` | modified (generated) | TS incremental build cache; do not commit / no behavior narrative | generated |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role | Doc |
|---|---|---|---|
| `docs/platforms/website/component-structure.md` | modified | Folder ownership (shared modals vs `customer/modals`) | structure §1.1 / §3 |
| `docs/platforms/website/flow-customer-meetings.md` | modified | Details modals contract + this inventory | all / §12.2 |
| `docs/platforms/website/flow-form-foundation.md` | modified | §3.8b FormModal + customer form modals | §3.8b, §5 |
| `.cursor/rules/website-customer-form-modal-placement.mdc` | added | Placement + forbidden Redux JSX factory | §6.4 |
| `.cursor/skills/website-customer-form-modal/SKILL.md` | added | Repeatable customer form-modal workflow | §6.4 |
| `.cursor/skills/website-customer-meeting-form/SKILL.md` | modified | Points details writes at form-modal skill/rule | skill §8 |

### 12.3 Meeting meta chips + notify status (this go-doc slice)

Exhaustive inventory for surfacing type/status/notify as distinct icon+color chips on cards and the details header (replacing the joined `·` meta line and the status·type subtitle).

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/meetings/MeetingMetaChips.tsx` | added | Presentational chips (icon + tone per value); `*Value` discriminators | §4.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingCard.tsx` | modified | Dropped `metaLine`; renders `MeetingMetaChips`; added `statusValue`/`typeValue`/`notifyStatusValue` props | §4.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingsScreen.tsx` | modified | Passes `*Value` + `notifyStatusLabel` to card | §4.1 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | modified | Header `Col` wraps `SectionHeading` + `MeetingMetaChips`; drops `headerSubtitle` | §6.2 |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | modified | Passes `*Value` to reused card; drops local `notifyLabel` gate | §4.1 |
| `src/app/ui/components/customer/hooks/useCustomerMeetings.ts` | modified | List GQL selects `notify_status { value label }` | §3 |
| `src/resources/configs/utils.ts` | modified | Added `stateSoftError`/`stateSoftWarning`/`stateSoftSuccess`/`stateSoftInfo` tokens | §4.2 |
| `lib/tsconfig.tsbuildinfo` | modified (generated) | TS incremental build cache; no narrative | generated |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role | Doc |
|---|---|---|---|
| `docs/platforms/website/flow-customer-meetings.md` | modified | §3 list GQL, §4.1 card, §4.2 chips, §6.2 header, this inventory | §3–§6, §12.3 |
| `docs/platforms/website/flow-customer-shell.md` | modified | Card reuse note (chips + `*Value`) | shell §196 |
| `.cursor/rules/website-meeting-meta-chips.mdc` | added | Distinct-chip + discriminator + soft-tint invariant | §4.2 |
| `.cursor/skills/website-semantic-color-audit/SKILL.md` | modified | Baseline map += four `stateSoft*` keys | audit map |

### 12.4 Home focus status card chips + hero percent (this go-doc slice)

Home command-hero follow-up: `CustomerHomeStatusCard` had remained on the old joined `metaLine` after directory/details chips shipped; hero percent used navy fill and vanished in dark mode. Ownership narrative: `flow-customer-shell.md` §7 / §12.1.

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/home/CustomerHomeStatusCard.tsx` | modified | `MeetingMetaChips` instead of `metaLine` | shell §7; §4.2 |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | modified | Passes focus `*Label`/`*Value` into status card | shell §7 |
| `src/app/ui/components/customer/home/HomeFocusVisual.tsx` | modified | Percent `fill` = `textPrimary` | shell §7 |
| `lib/tsconfig.tsbuildinfo` | modified (generated) | no narrative | generated |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role | Doc |
|---|---|---|---|
| `docs/platforms/website/flow-customer-shell.md` | modified | §7 ownership + §12.1 inventory | shell |
| `docs/platforms/website/flow-customer-meetings.md` | modified | this cross-ref inventory | §12.4 |
| `.cursor/rules/website-meeting-meta-chips.mdc` | modified | Home status card listed as consumer | §4.2 |
| `.cursor/rules/website-semantic-color-token-discipline.mdc` | modified | SVG content fill pairing | W43 |

### 12.5 Schedule policy, notes, and roster grouping (this go-doc slice)

Exhaustive inventory for the minimum lead, the notify edit freeze, `notify_start_at` derivation, demote-on-edit, voting-only quorum, the single alert-chrome component, and the datetime-picker honesty fix. Behavior: `meeting-domain.md` §3.2b / §9.1a, `meeting-live-state.md` §3.1, §6.5 above, `flow-form-foundation.md` §3.7.

#### Backend (`backend/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/orm/models/Meeting.ts` | modified | `NOTIFY_LEAD_MS` + `MIN_LEAD_MS` statics — single source for both windows | `meeting-domain.md` §3.2b |
| `src/app/orm/models/Customer.ts` | modified | `Ability.MEETING`: edit freeze on `update`, minimum lead on `approve`, quorum counts `CHAIRPERSON \| MEMBER` only | `meeting-domain.md` §9.1a–§9.1b |
| `src/app/validation/joi_rules.ts` | modified | `isMeetingDatetime(joi)` — lead rule on `create` / `update`, key `datetime.tooSoon` | `meeting-domain.md` §9.2 |
| `src/app/orchestrator/requesters/MeetingRequester.ts` | modified | `notify_start_at` derivation on create/update/approve; approve clears `live_state`; private `demoteApprovedMeetingToDraft` called by every `update`-family sub; `afterCommit` → `destroyMeetingLiveDoc(..., { flush: false })` | `meeting-domain.md` §9.1a, §9.3; `meeting-live-state.md` §3.1 |
| `src/resources/trans/ar/messages.ts` / `en/messages.ts` | modified | `MEETING_NOTIFY_TOO_SOON`, `MEETING_DATETIME_TOO_SOON`, voting-quorum wording on `MEETING_QUORUM_INCOMPLETE` | `meeting-domain.md` §9.5 |
| `src/resources/trans/ar/general.ts` / `en/general.ts` | modified | `joi.datetime.tooSoon` field message | `meeting-domain.md` §9.2 |
| `src/app/orchestrator/requesters/SubscriptionRequester.ts` | modified | Placement-only: `planItemName` moved from module scope into the namespace `private` section; no behavior change | `.cursor/rules/backend-requesters-governance.mdc` (Required Shape) |

Unchanged on purpose: SDL and generated GQL types — `_Meeting.notify_start_at` was already exposed, so this slice added no contract field and needed no `generate-types` run.

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/meetings/meetingScheduleLead.ts` | added | Mirror module: both windows, `meetingDatetimeMinDate`, `isVotingParticipantType`, `countVotingParticipants` | §6.5 |
| `src/app/ui/components/customer/meetings/MeetingNote.tsx` | added | The folder's single alert chrome (`title?` + `items`, bullets for 2+) | §6.2, §6.5 |
| `src/app/ui/components/customer/meetings/MeetingParticipantGroup.tsx` | added | Roster group heading + hairline divider | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | modified | Schedule-note derivation + dynamic title, `notify_start_at` line in basics, grouped roster, lock banner through `MeetingNote` | §6.2, §6.5 |
| `src/app/ui/components/customer/meetings/CustomerMeetingParticipantRow.tsx` | modified | `typeLabel` optional — the group heading carries the type | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingTemplateSlots.tsx` | modified | Contact-mode callout reuses `MeetingNote` (dropped the hand-rolled row and its hardcoded padding) | §6.2 |
| `src/app/ui/components/customer/meetings/CustomerMeetingFormScreen.tsx` | modified | `minDate` + `minDateError` on `datetime` | §5.1 |
| `src/app/ui/components/customer/modals/MeetingBasicsModal.tsx` | modified | Same `minDate` + `minDateError` on the edit path | §5.1, §6.4 |
| `src/app/ui/components/customer/hooks/useCustomerMeetingDetails.ts` | modified | Selects `notify_start_at` | §6.1 |
| `src/app/ui/components/customer/hooks/useCustomerHome.ts` | modified | `FocusReadiness.participantCount` → `votingCount` via `countVotingParticipants` | shell §7 |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | modified | Reads `votingCount` | shell §7 |
| `src/app/ui/components/form/FormDateTimeField.tsx` | modified | `minDateError` prop; min-lead rejection separated from past-date; stopped passing copy into the modal | form-foundation §3.7 |
| `src/app/ui/components/modals/DateTimePickerModal.tsx` | modified | Dropped `pastDateError` from props/op, no silent clamp or slot auto-advance, confirm disabled while the draft is invalid | form-foundation §3.7; W59 |
| `src/resources/translations/ar.ts` / `en.ts` | modified | `datetimeMinLeadError`, `scheduleNotesTitle*`, `scheduleNote*`, `participantsGroup*`; `templatesClear` → `templatesRemove`; removed `inviteStartLabel`; softened `datetimePastError` | §8 |
| `lib/tsconfig.tsbuildinfo` | generated | Touched by `yarn type-check`; reverted, never committed | generated |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-domain.md` | modified | §1, §2, §3.2/§3.2b, §9.1–§9.5, traceability |
| `docs/platforms/backend/contracts/meeting-live-state.md` | modified | §3.1 requester-driven reset; corrected the "no eviction / no callers" limit |
| `docs/platforms/website/flow-customer-meetings.md` | modified | §5.1, §5.3, §6.2, new §6.5, §7, this inventory |
| `docs/platforms/website/flow-form-foundation.md` | modified | §3.7 field/modal contract |
| `docs/platforms/website/flow-customer-shell.md` | modified | Voting-quorum wording on the hero ring |
| `docs/platforms/website/component-structure.md` | modified | Customer meetings folder members |
| `docs/invariants/backend.md` | modified | B26 time-window policy |
| `docs/invariants/website.md` | modified | W59 client mirrors a server gate |
| `.cursor/rules/meeting-lifecycle-approve-lock.mdc` | modified | Schedule gates, voting quorum, demotion + live-doc reset, `joi_rules.ts` glob |
| `.cursor/rules/meeting-live-state.mdc` | modified | Non-negotiable 5b (lifecycle reset clears BLOB + registry) and corrected ceiling wording |
| `.cursor/rules/backend-requesters-governance.mdc` | modified | Requester-local helpers live inside the namespace `private` section |
| `.cursor/rules/website-backend-policy-mirror.mdc` | added | Mirror-module placement + honesty rules |
| `.cursor/skills/website-customer-meeting-form/SKILL.md` | modified | Steps 4/4b/4c + canonical references |

### 12.6 Client-owned invite start, draft-only editing, and cancel (this slice)

Replaces the derived `notify_start_at` + edit-freeze + demote-on-edit model from §12.5. New policy: `MIN_LEAD_MS` 10 minutes, customer-chosen `notify_start_at` at least `NOTIFY_MIN_GAP_MS` (5 minutes) before `datetime`, editing allowed only while `DRAFT`, and a `cancel` sub as the escape hatch after approval. Template FKs moved to their own `updateTemplates` sub and their own registered modal. Behavior: `meeting-domain.md` §3.2b / §9.1a, `meeting-live-state.md` §3.1, §6.5 above.

Two open product decisions this slice deliberately did **not** take, recorded so the next change does not assume they were handled:

- `cancel` leaves `notify_status` untouched. Harmless today because no notify pipeline exists in `backend/src`; a future scheduler must filter on `status` as well.
- With `MIN_LEAD_MS` at 10 minutes, `ATTEND_OPEN_BEFORE_MS` (30 minutes) is structurally always open for a meeting created at the minimum lead — self-check-in effectively opens as soon as the meeting is approved.

#### Backend (`backend/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/orm/models/Meeting.ts` | modified | `MIN_LEAD_MS` 10 min; `NOTIFY_LEAD_MS` → `NOTIFY_MIN_GAP_MS` 5 min; module-tail self-check that throws when the gap reaches the lead | `meeting-domain.md` §3.2b |
| `src/app/orm/models/Customer.ts` | modified | `Ability.MEETING`: `update` requires `DRAFT`, `approve` checks `notify_start_at >= now`, new `cancel` sub | `meeting-domain.md` §9.1a–§9.1b |
| `src/app/validation/joi_rules.ts` | modified | New `isMeetingNotifyStartAt` reading its sibling `datetime` through `smartHelpers.get`; `isMeetingDatetime` no longer publishes a helper key | `meeting-domain.md` §9.2 |
| `src/app/orchestrator/requesters/MeetingRequester.ts` | modified | `notify_start_at` in create/read/update; template FKs split into the new `updateTemplates` sub; `demoteApprovedMeetingToDraft` removed; new `cancel` sub | `meeting-domain.md` §9.3 |
| `src/app/gql/definitions/customer.graphql` / `gql-types/customer.ts` | modified | `_Meeting.canCancel: _Ability` | `meeting-domain.md` §9.1 |
| `src/app/gql/bridges/customer/MeetingBridge.ts` | modified | `canCancel` in `MEETING_ABILITY_EXTRAS` + `loadExtra` | `meeting-domain.md` §9.1 |
| `requesters.website.ts` | modified | `customer.meeting` sub union gains `updateTemplates` and `cancel` | `meeting-domain.md` §9.3 |
| `src/resources/trans/{ar,en}/general.ts` | modified | `joi.notify_start_at.past` / `.tooLate` | `meeting-domain.md` §9.2 |
| `src/resources/trans/{ar,en}/messages.ts` | modified | Added `MEETING_ALREADY_STARTED`, `MEETING_NOTIFY_TIME_PASSED`; removed `MEETING_NOTIFY_TOO_SOON`, `MEETING_DATETIME_TOO_SOON` | `meeting-domain.md` §9.5 |

Copy discipline applied to the two new messages: `MEETING_ALREADY_STARTED` also fires for `COMPLETED` and for an already-canceled meeting, so it states only that cancel is no longer possible; `MEETING_NOTIFY_TIME_PASSED` names the invite start time, which is the field that failed, not the meeting time.

#### Website (`website/` repo)

| Path | Status | Role | Doc |
|---|---|---|---|
| `src/app/ui/components/customer/meetings/meetingScheduleLead.ts` | modified | `MEETING_MIN_LEAD_MS` 10 min; dropped `MEETING_NOTIFY_LEAD_MS` | §6.5 |
| `src/app/ui/components/customer/meetings/meetingNotifyTemplateMode.ts` | modified | Added the mirrored template families `MEETING_WHATSAPP_TEMPLATE_TYPES` / `MEETING_EMAIL_TEMPLATE_TYPES`, typed through `_MessageTemplateTypeValue` | §6.3, §6.5 |
| `src/app/ui/components/customer/meetings/CustomerMeetingFormScreen.tsx` | modified | `notify_start_at` datetime field on create, with the 5-minute gap stated in `subTitle` | §5.1 |
| `src/app/ui/components/customer/modals/MeetingBasicsModal.tsx` | modified | Same field and hint on the basics edit path | §6.4 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | modified | Freeze state removed; approval-final notes; `canCancel` action with `confirm()`; readiness card limited to `DRAFT` / `WAITING_TO_START` | §6.2, §6.5 |
| `src/app/ui/components/customer/hooks/useCustomerMeetingDetails.ts` | modified | Selects `canCancel { value }` | §6.1 |
| `src/app/ui/components/modals/DateTimePickerModal.tsx` | modified | `TIME_STEP_MINUTES` 15 → 5 so the 10-minute lead is actually reachable | form-foundation §3.7 |
| `src/types/gql/**`, `src/types/requesters/requesters.website.ts` | modified | Generated mirrors of the backend contract | data-flow §2 |
| `src/app/ui/components/customer/modals/MeetingTemplatesModal.tsx` | added | Registered `MEETING_TEMPLATES` modal: own `useShallowForm`, `read` on enter, two `messageTemplates` picker fields, save → `updateTemplates` | §6.4 |
| `src/resources/configs/store/modals.ts` | modified | `MEETING_TEMPLATES` identity + props type | §6.4 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` (templates) | modified | Dropped `meetingToUpdateValues` and the page-level template write; section Edit opens the modal | §6.3 |
| `src/app/ui/components/customer/meetings/CustomerMeetingTemplateSlots.tsx` | modified | Display-only slots (pick/clear callbacks and labels removed) | §6.3 |
| `src/app/ui/components/form/FormEntityPickerField.tsx` | modified | `clearLabel` prop — clear action for optional pickers | form-foundation §3.6 |
| `src/resources/translations/ar.ts` / `en.ts` | modified | `ui.modals.entityPicker.clear`, `templatesModalTitle` / `templatesModalSubtitle` (`templatesRemove` dropped), `notifyStartAtLabel` / `notifyStartAtSubtitle` / `notifyStartAtEmpty` / `notifyStartAtModalTitle` / `notifyStartAtPastError`, `scheduleNoteEditUntilApproval` / `scheduleNoteEditClosed`, `cancelMeetingButton` / `cancelMeetingConfirm`; removed the freeze/re-approval keys | §8 |
| `lib/tsconfig.tsbuildinfo` | modified | Build artifact from `type-check`; not narrated | — |

#### Root (`docs/` + `.cursor/`)

| Path | Status | Role |
|---|---|---|
| `docs/platforms/backend/contracts/meeting-domain.md` | modified | §1, §2, §3.2/§3.2b, §9.1–§9.5, traceability |
| `docs/platforms/backend/contracts/meeting-live-state.md` | modified | §3.1 reset triggers are now approve + cancel |
| `docs/platforms/website/flow-customer-meetings.md` | modified | §6.2–§6.5 rewrite + this inventory |
| `docs/platforms/website/flow-form-foundation.md` | modified | §3.6 entity-picker clear action; §3.7 picker step |
| `docs/platforms/website/component-structure.md` | modified | `MeetingTemplatesModal` in the customer modal folder listing |
| `docs/invariants/backend.md` | modified | B26 static rename |
| `.cursor/rules/meeting-lifecycle-approve-lock.mdc` | modified | Draft-only editing, cancel, invite-gap window, model self-check |
| `.cursor/rules/backend-requesters-governance.mdc` | modified | Dropped the demote-helper example |
| `.cursor/skills/website-customer-meeting-form/SKILL.md` | modified | Step 4b window wording |
| `.cursor/skills/website-customer-form-modal/SKILL.md` | modified | `MeetingTemplatesModal` listed as a canonical registered customer form modal |

## 13) Related

- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/website/flow-customer-members.md` (list/form pattern sibling)
- `docs/platforms/website/flow-form-foundation.md` (§3.8b registered customer form modals)
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `.cursor/rules/website-route-static-before-parametric.mdc`
- `.cursor/rules/website-customer-list-history-search.mdc`
- `.cursor/rules/website-result-lane-skeleton-shape.mdc`
- `.cursor/rules/website-customer-form-modal-placement.mdc`
- `.cursor/rules/website-backend-policy-mirror.mdc`
- `.cursor/rules/meeting-lifecycle-approve-lock.mdc`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
- `.cursor/skills/website-customer-form-modal/SKILL.md`
- `.cursor/skills/website-customer-meeting-form/SKILL.md`
