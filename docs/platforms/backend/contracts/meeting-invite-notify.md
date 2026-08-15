# Meeting Invite Notify Pipeline (Current)

Authoritative contract for the scheduled meeting-invite send path. Related: `meeting-domain.md` (lifecycle + `notify_start_at`), `meeting-participant-domain.md` (delivery columns), `message-template-domain.md` (copy + `lang`), `message-channel-domain.md` (credentials + `DISABLED`), `scheduler-console-seed-db.md` (task registration), `runtime-integrations.md` §6 (platform `MainEmail`).

Website chips: `docs/platforms/website/flow-customer-meetings.md` §4.2. Template `lang` UI: `flow-customer-message-templates.md`.

## 1) Scope

Shipped:

- Scheduler task `InviteNotifyTask` (registered in `cron.ts`), 6-field cron `*/5 * * * * *` (every 5 seconds). `cron-time-generator` has no `.seconds()`; `CronJob` already accepts seconds as the first field (`TaskBase` default is `* * * * * *`).
- Claim: `notify_status` `NOT_STARTED` → `WAITING_TO_NOTIFY` when meeting `status` is `WAITING_TO_START` or `STARTED` and `notify_start_at <= now`.
- Per-`MessageChannel` paced send queues plus one **global** platform-mail queue (`SystemSetting` key `ejtmaa_email_invite_next_at`). Not per organization.
- Token resolve + invite link in `MeetingInviteHelper` (no send in that file).
- Provider send: `sendAdwhatsMessage`, `sendAdwhatsProTemplate`, `sendCustomEmail`, platform `sendEmail("MainEmail")` inside the task.
- Discriminated send result `{ kind: "ok" | "death" | "recipient" }`.
- Participant axis arrays `delivered_channels` / `failed_channels` (`EMAIL` | `WHATSAPP`). No retry of an axis already on either array.
- Meeting `notify_status` finalize: `NOTIFIED` | `PARTIALLY_NOTIFIED` | `FAILED`.
- Cancel same transaction: leftover `PENDING` → `CANCELED`, then `finalizeNotifyStatus`. Keep `live_state: null` + `afterCommit` `destroyMeetingLiveDoc(..., { flush: false })`.
- Channel `DISABLED` (save/`testConnection` or provider death): fail leftover recipients on that channel’s axis, then finalize if done.
- GQL opt-out: `MessageChannelBridge.registerOrmAttrs.expect: ["invite_next_at"]` (cursor is not a customer field).

Approve still does **not** send and does **not** change `notify_status`.

Out of scope (not shipped):

- Starting notify inside `approve`.
- Supervisor / cpanel notify UI (`cpanel/` GQL checkout deferred — B14).
- New Twig / second platform email class.
- Retry of a failed axis.
- Recording an in-flight send that completes after `cancel` (accepted leftover — §8).

Schema apply: existing `DatabaseConsole` `alter` only.

## 2) Domain purpose

Invite progress is independent of live session lifecycle.

| Axis | Owner | Values |
|---|---|---|
| Session | `Meeting.status` | `DRAFT` \| `WAITING_TO_START` \| `STARTED` \| `COMPLETED` \| `CANCELED` |
| Invite clock | `Meeting.notify_start_at` | client-owned; approve does not re-derive |
| Invite progress | `Meeting.notify_status` | `NOT_STARTED` \| `WAITING_TO_NOTIFY` \| `NOTIFIED` \| `PARTIALLY_NOTIFIED` \| `FAILED` |
| Per-recipient | `MeetingParticipant.delivery_status` + channel arrays | see §4 |

Recipients: every roster row (`CHAIRPERSON` / `MEMBER` / `VIEWER`).

Intended axes (from meeting FKs, not from member contacts):

- `WHATSAPP` if `whatsapp_template_id` is set
- `EMAIL` if `email_template_id` is set

Missing contact on a required axis fails that axis only (`recipient`). The other axis and other members continue.

## 3) Persistence and config

### 3.1 Meeting (existing columns; enum expanded)

`notify_status` already exists on `meetings`. This contract owns the extra enum values (`PARTIALLY_NOTIFIED`, `FAILED`) and the send writer.

### 3.2 MeetingParticipant

| Column | Type | Notes |
|---|---|---|
| `delivery_status` | enum `meetingParticipantDeliveryStatus` | `PENDING` \| `SENT` \| `FAILED` \| `CANCELED`; default `PENDING` |
| `delivered_channels` | `ARRAY(STRING)` many-enum `meetingParticipantDeliveredChannel` | nullable; success only |
| `failed_channels` | same | nullable; failure only |
| `notified` | BOOLEAN | unchanged; task does not write it |

`SENT` if at least one intended axis is on `delivered_channels`. `FAILED` only when every intended axis is on `failed_channels` and none succeeded. `CANCELED` is written by `MeetingRequester.cancel` on leftover `PENDING` only; `SENT` / `FAILED` rows stay.

### 3.3 MessageTemplate

| Column | Type | Notes |
|---|---|---|
| `lang` | enum `messageTemplateLang` | `AR` \| `EN`; `allowNull: false`; `defaultValue: "AR"` |
| `meta_template_internal_name` | STRING(191) | Pro provider template name; existing Pro rows without it fail send until the customer re-saves |

Pro write re-reads the approved list and overwrites `lang` (`ar` → `AR`, `en` → `EN`) plus `meta_template_internal_name`. Client `lang` is not trusted for Pro. List filter keeps only `language === "ar"` or `"en"` (reject `en_US` and every other code).

### 3.4 MessageChannel

| Column | Type | Notes |
|---|---|---|
| `invite_next_at` | DATE | nullable pace cursor; **not** on GQL (`expect`) |

`EJTMAA_EMAIL` has no channel row. Platform mail uses `SystemSetting.ejtmaa_email_invite_next_at` (ISO string in `ValuesMap`). Not on VAT / website-settings forms.

### 3.5 Localization

`backend/src/resources/trans/{ar,en}/general.ts`:

- `messageTemplateLang`: `AR`, `EN`
- `meetingNotifyStatus`: + `PARTIALLY_NOTIFIED`, `FAILED`
- `meetingParticipantDeliveryStatus`: + `CANCELED`
- `meetingParticipantDeliveredChannel`: `EMAIL`, `WHATSAPP` (many-enum; not a combined `EMAIL_WHATSAPP` value)

## 4) Finalize rules

Exported `IMeetingRequester.finalizeNotifyStatus(meeting, participants)`:

1. Intended axes from the two template FKs. If none → `FAILED`.
2. `anySuccess` if any participant `delivery_status === "SENT"`.
3. `allIntendedSucceeded` if every participant has every intended axis in `delivered_channels`.
4. No success → `FAILED`. All succeeded → `NOTIFIED`. Else → `PARTIALLY_NOTIFIED`.

Cancel-before-any-send therefore finalizes `FAILED`. There is no `CANCELED` value on `notify_status`.

## 5) Scheduler

File: `backend/src/app/scheduler/tasks/InviteNotifyTask.ts`  
Register: `backend/src/resources/configs/scheduler/cron.ts` (`InviteNotifyTask` next to `ExpireSubscriptionsTask`).

### 5.1 Tick

Module-level `running` (TaskBase has no overlap flag). One process; overlapping ticks return immediately.

1. `claimDueMeetings` — conditional bulk `UPDATE`.
2. `loadNotifyMeetings` — `status IN (WAITING_TO_START, STARTED)` and `notify_status === WAITING_TO_NOTIFY`, include org + templates + channels + participants + members.
3. Build ready queues; one attempt per queue per tick; pace after a send attempt.

`COMPLETED` and `CANCELED` meetings are not loaded.

### 5.2 Queues

One queue per distinct `MessageChannel` used by loaded meetings:

- `ACTIVE` and `invite_next_at` due (or null) → send path
- `DISABLED` → no send; `failRemainingOnQueue` on that axis (same leftover fail as provider death)

`CUSTOM_EMAIL` uses the email template’s channel. WhatsApp uses the WhatsApp template’s channel (`ADWHATS` / `ADWHATS_PRO`).

Platform email: if any loaded meeting’s email template `type === EJTMAA_EMAIL` and the global cursor is due → one `platform-email` queue for the whole app.

Pacing after a send (or death) attempt:

| Queue | Delay |
|---|---|
| `ADWHATS` | 10000–14999 ms (product jitter) |
| `ADWHATS_PRO` | 5000 ms |
| `CUSTOM_EMAIL` / `EJTMAA_EMAIL` | 20000 ms |

`DISABLED` leftover-fail does not pace.

### 5.3 Work selection

Per queue, first meeting that uses that channel (or platform email), then first participant (sorted by `member_id`) that still `needsAxis`: not `CANCELED`, and axis not already in `delivered_channels` or `failed_channels`.

### 5.4 Send then write

Provider call is **outside** the write transaction. Then one transaction:

1. Reload meeting + participant.
2. `death` on a channel queue → set channel `DISABLED`, `failRemainingOnQueue`, pace, return.
3. Write the axis only if meeting status is `WAITING_TO_START` or `STARTED` **and** participant is not `CANCELED`.
4. If all intended axes for all non-canceled participants are decided → `notify_status = finalizeNotifyStatus`.
5. Pace the queue.

`failRemainingOnQueue` reloads notify meetings in the same transaction, writes `recipient` fail on leftover needs, and finalizes meetings that are still `WAITING_TO_START` or `STARTED` when axes are done.

### 5.5 Tokens and link

`MeetingInviteHelper`:

- Link: `https://{custom_domain || subdomain.ejtmaa.live}/meeting/{memberId}/{access_token}/{meetingId}`. Literal `ejtmaa.live` matches website `ORG_PUBLIC_DOMAIN`. If both host fields are empty, host is `ejtmaa.live`.
- `{{meetingDateTime}}`: `moment-timezone` `Asia/Riyadh`, locale from template `lang`, `dddd D MMMM YYYY` + `hh:mm A`.
- Tokens: `memberName`, `meetingSubject`, `meetingDateTime`, `meetingLink`. Unknown `{{tokens}}` left as-is.

Join CTA title: `دخول الاجتماع` / `Join meeting` from template `lang`.

### 5.6 Provider results

| Helper | Death | Recipient |
|---|---|---|
| `sendAdwhatsMessage` | account not ready, or HTTP 401/403 | other send errors; empty mobile |
| `sendAdwhatsProTemplate` | account not ready, or HTTP 401/403 | missing draft id / internal name; empty mobile; other send errors |
| `sendCustomEmail` | `verifyCustomEmailConnection` false | `sendEmail` throw; empty email |
| platform `MainEmail` in the task | not used (catch → `recipient`) | missing template/email or `sendEmail` throw |

`isInviteSendAuthDeath`: Axios 401/403 only.

Custom SMTP send uses `sendEmail("MainEmail")` with EmailBase `host` / `senderAccount` refs (not raw `sendMail`). Optional `fromName` / `fromAddress` / `link`.

Pro TEMPLATE envelope: body slots from stored `variables` map, sorted by slot number, text from invite tokens.

Empty `meta_template_internal_name` on an old Pro row → `recipient` (no send).

## 6) Ability locks that protect the pipeline

`Customer.can("MESSAGE_CHANNEL")`:

- `delete`: any type, if any `MessageTemplate` has this `message_channel_id` → `CANNOT_DELETE_USED`.
- `update`: `CANNOT_UPDATE_USED` only when `type === ADWHATS_PRO` and template count > 0. Classic Ad Whats and custom email may still save (and `testConnection` may flip `DISABLED`).

`MESSAGE_TEMPLATE` `delete` remains blocked when a meeting FK still points at the template.

`MEETING` `cancel` stays allowed for `DRAFT` or `WAITING_TO_START` and does **not** check `notify_status`.

## 7) GraphQL

`base.graphql`: `_MessageTemplateLang` / `Value`; `_MeetingNotifyStatusValue` + `PARTIALLY_NOTIFIED` `FAILED`; `_MeetingParticipantDeliveryStatusValue` + `CANCELED`; `_MeetingParticipantDeliveredChannel` / `Value`.

`customer.graphql`:

- `_MessageTemplate.lang`, `meta_template_internal_name`
- `_MeetingParticipant.delivered_channels`, `failed_channels`
- `_AdwhatsProApprovedTemplate.language` (provider scalar `ar` / `en` after filter)

`invite_next_at` is not on `_MessageChannel`.

Codegen: backend `yarn generate-types`. Website mirrors `base` + `customer` SDL and gql-types. Supervisor generated `gql-types/supervisor.ts` updates because shared `base` changed; no supervisor UI. `cpanel/` GQL mirror files were not present in this workspace (B14 deferred).

## 8) Accepted leftover — cancel during send

Cancel already writes leftover `PENDING` → `CANCELED` and finalizes `notify_status` in the same transaction.

The task sends first, then reloads. If the meeting or row is already canceled, it does **not** write the axis. Typically one in-flight message (one attempt per queue per tick). If WhatsApp and email queues are both due in the same tick, at most two. After that tick the meeting is `CANCELED` and is not loaded again.

Product decision: leave this race. Consequence: a recipient may receive an invite to a canceled meeting; the row stays `CANCELED`; `notify_status` was already finalized without that success. The pipeline does not stick.

## 9) Failure / empty modes

| Condition | Behavior |
|---|---|
| No intended template FKs | `finalizeNotifyStatus` → `FAILED` |
| Channel `DISABLED` | leftover axis → `failed_channels`; finalize when done |
| Provider auth death 401/403 or not ready | channel `DISABLED` + leftover fail |
| Missing mobile/email on a required axis | that axis `recipient` fail; other axis continues |
| Pro row missing `meta_template_internal_name` | that send `recipient` |
| Meeting `STARTED` during send | still loaded and written (accepted statuses are waiting or started) |
| Meeting `CANCELED` / `COMPLETED` | not loaded; in-flight write skipped if status no longer live |
| Overlapping tick | `running` → return |
| Platform mailer throw | `recipient` (does not disable a channel) |

## 10) Website surfaces

- `MeetingMetaChips`: `PARTIALLY_NOTIFIED` → `FiAlertCircle` / warning; `FAILED` → `FiXOctagon` / danger. Labels from backend enum.
- Template form: `FormChoiceField` `name="lang"` (`AR`/`EN`). Create default `AR`. Free-text kinds editable. Pro `readOnly`; value from picker `language` (`ar`/`en` only). Not `LanguageSwitch`, not `FormTextField`.
- Entity picker selection may carry `language`; `getLanguage` on `adwhatsProApprovedTemplates`; field/modal must not strip it.

## 11) Verification

Existing scripts only:

- Backend: `yarn type-check` (required after task/Ability/helper edits). `yarn generate-types` when SDL changes.
- Website: `yarn type-check` after GQL mirror / form / chip edits.

## 12) Traceability map

| Path | Role | § |
|---|---|---|
| `backend/src/app/scheduler/tasks/InviteNotifyTask.ts` | Claim, queues, send, leftover fail, finalize | §5 |
| `backend/src/resources/configs/scheduler/cron.ts` | Task registration | §5 |
| `backend/src/app/helpers/MeetingInviteHelper.ts` | Link, tokens, datetime | §5.5 |
| `backend/src/app/helpers/AdWhatsDevApi.ts` | Classic send + death | §5.6 |
| `backend/src/app/helpers/AdWhatsProDevApi.ts` | Pro TEMPLATE send + death | §5.6 |
| `backend/src/app/helpers/CustomEmailHelper.ts` | Custom SMTP send via `MainEmail` refs | §5.6 |
| `backend/src/app/helpers/CustomerAdwhatsLists.ts` | Approved-list `ar` / `en` filter | §3.3 |
| `backend/src/app/mailer/emails/MainEmail.ts` | Optional `host` / `senderAccount` | §5.6; `runtime-integrations.md` §6 |
| `backend/src/app/orchestrator/requesters/MeetingRequester.ts` | `cancel` leftover + `finalizeNotifyStatus` | §4, §6 |
| `backend/src/app/orchestrator/requesters/MessageTemplateRequester.ts` | `lang` + Pro overwrite | §3.3 |
| `backend/src/app/orm/models/Customer.ts` | Channel delete/update locks | §6 |
| `backend/src/app/orm/models/MeetingParticipant.ts` | Delivery arrays + `CANCELED` | §3.2 |
| `backend/src/app/orm/models/MessageChannel.ts` | `invite_next_at` | §3.4 |
| `backend/src/app/orm/models/MessageTemplate.ts` | `lang` + `meta_template_internal_name` | §3.3 |
| `backend/src/app/orm/models/SystemSetting.ts` | `ejtmaa_email_invite_next_at` | §3.4 |
| `backend/src/app/gql/bridges/customer/MessageChannelBridge.ts` | `expect: ["invite_next_at"]` | §7 |
| `backend/src/app/gql/definitions/base.graphql` | Shared notify / lang / channel enums | §7 |
| `backend/src/app/gql/definitions/customer.graphql` | Template lang, participant arrays, Pro `language` | §7 |
| `backend/src/app/gql/gql-types/*.ts` | Generated | generated |
| `backend/src/resources/trans/ar/general.ts` / `en/general.ts` | Enum labels | §3.5 |
| `website/src/app/ui/components/customer/meetings/MeetingMetaChips.tsx` | Notify chips | §10; `flow-customer-meetings.md` §4.2 |
| `website/src/app/ui/components/customer/message-templates/CustomerMessageTemplateFormScreen.tsx` | `lang` field | §10; `flow-customer-message-templates.md` |
| `website/src/app/ui/components/form/FormEntityPickerField.tsx` | Persist picker `language` | §10 |
| `website/src/app/ui/components/modals/EntityPickerModal.tsx` | Confirm keeps `language` | §10 |
| `website/src/app/ui/components/modals/entity-picker/configs/adwhatsProApprovedTemplates.tsx` | `getLanguage` | §10 |
| `website/src/types/gql/definitions/*.graphql` | Website SDL mirror | §7 |
| `website/src/types/gql/gql-types/*.ts` | Generated mirror | generated |

`cpanel/` GQL mirrors are deferred (B14).

## Related

- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/meeting-participant-domain.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/contracts/message-channel-domain.md`
- `docs/platforms/backend/patterns/scheduler-console-seed-db.md`
- `docs/platforms/backend/modules/runtime-integrations.md`
- `docs/invariants/backend.md` (B27)
- `docs/platforms/website/flow-customer-meetings.md`
- `docs/platforms/website/flow-customer-message-templates.md`
- `.cursor/rules/meeting-invite-notify.mdc`
