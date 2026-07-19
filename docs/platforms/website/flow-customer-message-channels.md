# Website Flow — Customer Message Channels

Authenticated customer org delivery-channel directory + multi-path form on `CUSTOMER_MAIN`. Shell/breadcrumb contract: `flow-customer-shell.md` §7.1. Backend contract: `docs/platforms/backend/contracts/message-channel-domain.md`. Form foundation: `flow-form-foundation.md` (§3.8 ConfirmModal, multi-action loading).

## 1) Scope

**Shipped**

- Directory (GQL list + ResultLane load-more) with Add + card Edit.
- Multi-path form `CustomerMessageChannelForm` (create/update/delete) via `Forms.CUSTOMER_MESSAGE_CHANNEL` → `messageChannel` requester.
- Drawer tile `itemMessageChannels` / `CustomerMessageChannels` / **`FiSend`**, after Subscription / before Settings.
- Card glyph **`FiSend`** (section identity consistency); presentational card with optional `editLabel` / `onEdit`.
- Type chooser on **create** only; update shows locked type label (value kept in form from `read`).
- Status **not** on the form: create/update set `ACTIVE` / `DISABLED` via requester `testConnection()` (model stub currently returns `false` → saves land as `DISABLED` until real connectivity ships).
- Conditional fields by `type`: `CUSTOM_EMAIL` SMTP block vs `ADWHATS` / `ADWHATS_PRO` token + account id (type-dependent Ad Whats labels).
- Delete: `await confirm(…, "danger")` → `sub: "delete"` → `nav.back()`.
- Per-sub button loading: `saving` for `create`/`update`, `deleting` for `delete` (from `currentSub`), not a blanket spinner on Save during delete.
- User-facing section copy may say **WhatsApp**; card type labels come from backend enum (أد واتس / Ad Whats).

**Not shipped**

- History search / type-status filter chips (no `_MessageChannelFilter` yet).
- Calling `testConnection()` from the UI (requester-owned only).
- Channel details route.
- Seed rows for channels.

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| List route | `CustomerMessageChannels` → `/customer/message-channels` |
| Form route | `CustomerMessageChannelForm` → `/customer/message-channels/form` (+ `/:id`) |
| List page | `website/src/app/ui/pages/customer/CustomerMessageChannels.tsx` |
| Form page | `website/src/app/ui/pages/customer/CustomerMessageChannelForm.tsx` (thin `MyPage`) |
| List screen | `…/message-channels/CustomerMessageChannelsScreen.tsx` |
| Form screen | `…/message-channels/CustomerMessageChannelFormScreen.tsx` |
| Card | `…/message-channels/CustomerMessageChannelCard.tsx` |
| Skeleton | `…/message-channels/MessageChannelCardSkeleton.tsx` |
| Hook | `…/hooks/useCustomerMessageChannels.ts` |
| Href helper | `buildCustomerMessageChannelFormHref` in `resources/configs/customer/formRoute.ts` |
| Form registry | `Forms.CUSTOMER_MESSAGE_CHANNEL` → `API.FORMS.CUSTOMER.R("messageChannel")` |
| Confirm | `confirm()` from `modals/ConfirmModal.tsx` (`Modals.CONFIRM`) |
| Drawer | `CustomerDrawer.tsx` |

List + form: `layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`. List breadcrumb parent `CustomerHome`; form parent `CustomerMessageChannels` with param-aware create/edit label. Form identify registered **after** list on the same prefix (`.cursor/rules/website-route-static-before-parametric.mdc`).

## 3) Form contract

`formType = id ? "update" : "create"` from `useCurrentParams<"CustomerMessageChannelForm">` — no parallel `isUpdate` flag. Never branch on page identify.

| Mode | Behavior |
|---|---|
| create | Stable `formIdentify` (`customer-message-channel-form-create`) + `removeOnExit: false`; **`d.reset()` on create success** before `nav.back()` |
| update | `initProps.values: { messageChannel: id }`; `didEntered` → `send({ sub: "read" })`; `Loadable` while `!exist` or read in flight; `removeOnExit` true (default generate identify). Do **not** echo `messageChannel` id from requester `read` — keep id from `initProps` |
| header | `SectionHeading` + back + primary Save (+ neutral Delete on update) — actions not full-width under fields |
| delete | `await confirm(t("deleteConfirm"), "danger")` → `sub: "delete"` → `nav.back()` |
| loading | `formLoading` = any `SENDING`; `saving` = `SENDING` && (`create`\|`update`); `deleting` = `SENDING` && `delete` |
| toast | Automatic only — no manual toast in `afterSentSuccess` |

### 3.1 Fields

| Field | Create | Update | Notes |
|---|---|---|---|
| `name` | yes | yes | |
| `type` | `FormChoiceField` | read-only label | Update must still submit echoed `type` from `read` (backend locks) |
| `smtp_*` / `from_*` | when `CUSTOM_EMAIL` | same | `smtp_secure` as `"true"` / `"false"` choice tiles |
| `adwhats_token` / `adwhats_account_id` | when `ADWHATS` \| `ADWHATS_PRO` | same | Labels switch for Pro |
| `status` | not on UI | not on UI | Server-owned |

## 4) Directory data adapter

Mount-private `"customer-message-channels"` inheriting `DATA_ADAPTERS.CUSTOMER_GQL`, `listable: "messageChannels"`. List selection: name / type / status / `from_address` | `adwhats_account_id` (no secrets required for the card). Add → `buildCustomerMessageChannelFormHref("create")`; Edit → `…("update", id)`.

## 5) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.messageChannels.*` | list title, subtitle, add, edit, empty, load-more |
| `ui.pages.customer.messageChannelForm.*` | create/edit titles, field labels, type options, Ad Whats labels, delete confirm, buttons |
| `ui.modals.confirm.*` | shared confirm title / confirm / cancel |
| `ui.components.mainHeader.back` | form back control |

ar/en mirrors required.

## 6) Failure / empty modes (UI)

| Mode | Behavior |
|---|---|
| Initial injection / update `read` | Header actions hidden; centered `Loadable` |
| List empty | ResultLane empty title/subtitle |
| List error | ResultLane retry |
| Delete cancel | `confirm` resolves `false` — no send |
| Validation / ability | Form middleware / toast — server-owned |

## 7) Traceability map (this change set)

| Path | Role | Section |
|---|---|---|
| `website/…/CustomerMessageChannelForm.tsx` | Thin page | §2 |
| `website/…/CustomerMessageChannelFormScreen.tsx` | Form screen | §3 |
| `website/…/CustomerMessageChannelsScreen.tsx` | List Add/Edit wiring | §4 |
| `website/…/CustomerMessageChannelCard.tsx` | Presentational card + edit | §1, §4 |
| `website/…/MessageChannelCardSkeleton.tsx` | Lane skeleton (unchanged pattern) | §4 |
| `website/…/modals/ConfirmModal.tsx` | Shared confirm | §3, `flow-form-foundation.md` §3.8 |
| `website/…/form/FormActionButton.tsx` | `tone="danger"` | `flow-form-foundation.md` |
| `website/…/members/CustomerMemberFormScreen.tsx` | Same confirm + loading pattern | `flow-customer-members.md` |
| `website/…/meetings/CustomerMeetingFormScreen.tsx` | `d.reset()` on create success | `flow-customer-meetings.md` |
| `website/src/resources/configs/routes.ts` | Multi-path form route | §2 |
| `website/src/resources/configs/customer/formRoute.ts` | Href builder | §2 |
| `website/src/resources/configs/store/forms.ts` | `CUSTOMER_MESSAGE_CHANNEL` | §2 |
| `website/src/resources/configs/store/modals.ts` | `CONFIRM` | §3 |
| `website/src/resources/translations/ar.ts` / `en.ts` | i18n | §5 |
| `website/src/types/gql/definitions/customer.graphql` | Credential fields on `_MessageChannel` | backend §4 + mirror |
| `website/src/types/gql/gql-types/customer.ts` | Generated types | generated |
| `website/src/types/requesters/requesters.website.ts` | `customer.messageChannel` | W18 |
| `website/lib/tsconfig.tsbuildinfo` | Build cache | **exclude from narrative** (generated noise) |

Backend paths for the same slice: `message-channel-domain.md` §10.

## 8) Related

- `.cursor/skills/website-customer-message-channels/SKILL.md`
- `.cursor/skills/website-customer-member-form/SKILL.md`
- `.cursor/rules/website-multi-path-form-routes.mdc`
- `.cursor/rules/website-confirm-modal.mdc`
- `.cursor/rules/message-channel-domain.mdc`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/backend/contracts/message-channel-domain.md`
