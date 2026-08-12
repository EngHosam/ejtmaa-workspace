# Message Template Domain Contract (Current)

## 1) Scope

Current Ejtmaa message-template surface:

- ORM persistence for organization-owned reusable notification templates,
- delivery kinds `EJTMAA_EMAIL` | `CUSTOM_EMAIL` | `ADWHATS` | `ADWHATS_PRO`,
- optional FK to `MessageChannel` (`message_channel_id` null only for `EJTMAA_EMAIL`),
- content columns by kind (`subject` / `body` / `variables`),
- customer GraphQL **read** of templates for the authenticated customer's organization,
- website GQL mirrors for that customer surface (`base` + `customer` SDL + gql-types),
- website write path via `MessageTemplateRequester` (`read` | `create` | `update` | `delete`) + `Customer.Ability.MESSAGE_TEMPLATE`.

Out of scope (not shipped):

- supervisor MessageTemplate GraphQL,
- cpanel mirrors/UI,
- seed rows for templates,
- nested `_Organization.messageTemplates` (use root list),
- inverse `MessageChannel.hasMany(MessageTemplate)` (template `belongsTo` only; see §3.4),
- renaming Meeting FKs `whatsapp_template_id` / `email_template_id` (still point at `MessageTemplate` rows; column names are legacy).

Related: `message-channel-domain.md` (delivery credentials). Meeting optional template FKs: `meeting-domain.md`. Website UI: `docs/platforms/website/flow-customer-message-templates.md`.

## 2) Domain purpose

`MessageTemplate` is a **non-actor** reusable message library row inside an `Organization`.

| `type` | `message_channel_id` | Content (write-path product rules; not ORM-enforced) |
|---|---|---|
| `EJTMAA_EMAIL` | null (platform mailer) | `subject` + `body` required on write |
| `CUSTOM_EMAIL` | required, channel `type=CUSTOM_EMAIL` | `subject` + `body` required on write |
| `ADWHATS` | required, channel `type=ADWHATS` | `body` only (`subject` null) |
| `ADWHATS_PRO` | required, channel `type=ADWHATS_PRO` | `meta_template_id` + `meta_template_label` + `variables` map (no free subject/body) |

- Tenant boundary is `organization_id` (not `customer_id` directly).
- Templates are referenced by meetings via optional FKs; meetings do not store inline template text.
- ORM does **not** enforce per-type nullability; write-path Joi/requester does.
- Credentials never live on the template row — only on `MessageChannel` (or the platform mailer for `EJTMAA_EMAIL`).

`variables` shape (Ad Whats Pro): `Record<string, string>` — fixed meeting-notify keys → Meta/Ad Whats Pro positional slot (`"1"` … `"n"`). Stored as JSONB (`isTypedObject: true`).

### Meeting-notify placeholders (subject / body / Ad Whats free text)

Resolved at send time with non-destructive substitution (unknown tokens left as-is):

| Token | Meaning |
|---|---|
| `{{memberName}}` | Recipient member display name |
| `{{meetingSubject}}` | Meeting subject |
| `{{meetingDateTime}}` | Localized meeting datetime |
| `{{meetingLink}}` | Per-recipient meeting join URL |

Website:

- Free-text kinds: human-labeled insert chips **above** the subject/body field (`headerArea`), not beside the title.
- `ADWHATS_PRO`: required `meta_template` picker (`adwhatsProApprovedTemplates`, gated on channel id) + `FormMessageTemplateVariablesField` after a template is selected — human labels only (no `{{token}}` in UI); default variable numbers `1`–`4`.

**Retired (do not reintroduce):** column/enum `channel` / `messageTemplateChannel` / values `WHATSAPP` | `EMAIL`; GQL `_MessageTemplateChannel*`.

## 3) ORM model

File: `backend/src/app/orm/models/MessageTemplate.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `messageTemplate`
- `tableName`: `message_templates`

### 3.1 Attrs layout

- `//relations` — `organization_id`, `message_channel_id`
- `//info` — `name`, `type`, `subject`, `body`, `variables`, `meta_template_id`, `meta_template_label`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment; Attrs typed `string` |
| `organization_id` | BIGINT | no | FK → `organizations.id` |
| `message_channel_id` | BIGINT | yes | FK → `message_channels.id`; null for `EJTMAA_EMAIL` |
| `name` | STRING(191) | no | library label |
| `type` | STRING(191) | no | enum metadata `messageTemplateType`; typed object |
| `subject` | STRING(191) | yes | email kinds |
| `body` | TEXT | yes | email / Ad Whats |
| `variables` | JSONB | yes | Ad Whats Pro map; typed object |
| `meta_template_id` | STRING(191) | yes | Ad Whats Pro only; approved template id; write-required when `type=ADWHATS_PRO` |
| `meta_template_label` | STRING(191) | yes | display name snapshot for the selected template |

Exported TS types:

- `MessageTemplateType = keyof G_Tr["enums"]["messageTemplateType"]`
- `MessageTemplateVariables = Record<string, string>`

### 3.3 Indexes

- `message_templates_organization_id`
- `message_templates_organization_id_type` (replaces legacy `…_channel`)
- `message_templates_message_channel_id`

### 3.4 Relations

`MessageTemplate.boot()`:

- `belongsTo(Organization)` on `organization_id`
- `belongsTo(MessageChannel)` on `message_channel_id`

`Organization.boot()` inverse:

- `hasMany(MessageTemplate)` on `organization_id`
- association mixins use PK type `string` (aligned with MessageChannel mixins)

**No** `hasMany(MessageTemplate)` on `MessageChannel`:

- avoids circular import (`MessageTemplate` already imports `MessageChannel`),
- no GQL field `_MessageChannel.messageTemplates`,
- no current read path that loads templates from the channel parent.

GQL nests `messageChannel` via template `belongsTo` only.

### 3.5 Localization

Enum key `messageTemplateType` under:

- `backend/src/resources/trans/ar/general.ts`
- `backend/src/resources/trans/en/general.ts`

| Value | EN label | AR label |
|---|---|---|
| `EJTMAA_EMAIL` | Platform email | بريد المنصة الإلكتروني |
| `CUSTOM_EMAIL` | Custom email | بريد إلكتروني مخصص |
| `ADWHATS` | Ad Whats | اد واتس |
| `ADWHATS_PRO` | Ad Whats Pro | اد واتس برو |

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MessageTemplateTypeValue` / `_MessageTemplateType`
- `backend/src/app/gql/definitions/customer.graphql` — `_MessageTemplate` + roots

### Type `_MessageTemplate`

Implements `_Timestamps` & `_Pagination`.

Fields: `id`, `name`, `type`, `subject`, `body`, `variables` (`JSONObject`), `meta_template_id`, `meta_template_label`, timestamps, `organization`, `messageChannel`, `total_count`.

### Root queries

- `messageTemplates(filter: _MessageTemplateFilter): [_MessageTemplate]` — optional `type` or `types[]`
- `messageTemplate(id: ID!): _MessageTemplate`

Resolvers (`CustomerSchema`): `prepareManyGQLModels({ me: true, filter })` / `prepareOneGQLModel({ me: true, id })`. Bridge maps `filter.type` or `filter.types` (`Op.in`).

`CustomerSchema.registeredBridges` includes `MessageTemplateBridge`.

### Helper root (Ad Whats Pro approved templates — not a Bridge)

`adwhatsProApprovedTemplates(filter: { messageChannelId })` in `customer.graphql`.

Helper: `CustomerAdwhatsLists.getCustomerAdwhatsProApprovedTemplates` → `AdWhatsProDevApi.listAdwhatsProApprovedTemplates`.

Requires an org-owned `MessageChannel` with `type === ADWHATS_PRO` and non-empty `adwhats_token` + `adwhats_account_id`; otherwise `[]`. Remote failure → `[]`. `total_count` = returned list length. No new Ability keys. `meta_template_id` stores the Pro draft id; `meta_template_label` stores the display name.

HTTP: `GET https://api.adwhats.com.sa/templates?whatsapp_account_id=` with header `token`. That call currently has **no** Axios timeout (account lists in `AdWhatsProDevApi` do).

### Bridge: `MessageTemplateBridge`

File: `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts`

- Extends `CustomerOrganizationOwnedBridgeBase`
- `ident = "messageTemplate"`, `typeIdent = "_MessageTemplate"`, `ormModel = MessageTemplateModel`
- `GetManyParent = OrganizationOwnedMeParent` (`{ me: true }`)
- `GetOneParent = MessageTemplateModel | MeetingModel | { me: true; id: string }`
  - `MeetingModel` for nested `_Meeting.whatsappTemplate` / `emailTemplate`

### Shared base: `CustomerOrganizationOwnedBridgeBase`

File: `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts`

On `{ me: true }`: require `context.customer`, load `getOrganization()`, return Organization or throw `NOT_PERMIT` / `404`.

### Inverse relation parent typing (mandatory)

| Nested field | Prepared bridge | Parent type that must be on `GetOneParent` |
|---|---|---|
| `_MessageTemplate.organization` | `OrganizationBridge` | `MessageTemplateModel` |
| `_MessageTemplate.messageChannel` | `MessageChannelBridge` | `MessageTemplateModel` |
| `_Meeting.whatsappTemplate` / `emailTemplate` | `MessageTemplateBridge` | `MeetingModel` |

`MessageChannelBridge.GetOneParent` therefore includes `MessageTemplateModel`.

## 5) Read flow (root)

### `messageTemplates`

1. `MessageTemplateBridge.AsRoot` → `prepareManyGQLModels({ me: true })`
2. `CustomerOrganizationOwnedBridgeBase.getRootOrmParent` → customer's Organization
3. Base list options: `withListable` + `withReplacements` + `order updated_at DESC`
4. Association load via Organization → message templates

### `messageTemplate(id)`

1. `prepareOneGQLModel({ me: true, id })`
2. Same org resolve as above
3. Base one options: `where: { id }`
4. Scoped to that organization association

## 5b) Customer write path

- Ability on `Customer`: `MESSAGE_TEMPLATE` with `sub: "create" | "read" | "update" | "delete"` and optional `messageTemplate` target.
  - `create`: customer must have an organization (`ACTION_NOT_ALLOWED` otherwise).
  - `read` / `update` / `delete`: resolve template; must belong to that organization (`404` missing, `NOT_PERMIT` other org).
- Joi helper: `isCustomerOwnedMessageTemplate` in `joi_rules.ts`.
- Requester: `MessageTemplateRequester` (`@requester("messageTemplate")`) — `read` | `create` | `update` | `delete` for website/customer.
- Website map: `customer.messageTemplate` in `requesters.website.ts` (mirrored on website).
- **`read` values must not echo `messageTemplate` id** — identity stays in form `initProps.values`.
- Update **locks** `type` to the existing row (client must echo the read value).
- Channel FK: null for `EJTMAA_EMAIL`; otherwise org-owned `MessageChannel` whose `type` matches template `type`.
- Content columns required by `type` via Joi `when`; unused branches `joi.any().optional().allow(null, "").strip()`. Persist with one flat write (`props.field ?? null`) — **no** per-type `attrsForType` helper. See `.cursor/rules/requester-type-conditional-strip.mdc`.
- Do **not** use `forbidden()` for leftover form content fields unless the client guarantees the key is absent; prefer `.strip()`.
- Channel write for credentials uses the same strip + `?? null` pattern (`MessageChannelRequester`).
- Form field `meta_template` is a SelectOption (`joi.select({ raw: true })` when `ADWHATS_PRO`). Write splits to `meta_template_id` (`${value}`) + `meta_template_label` (`label`). Empty picker is `null`. Pro writes `body: null`.
- Template picker list filter: website always requests `status: ACTIVE`. Failed `testConnection()` leaves the channel `DISABLED`, so the picker can look empty until the channel is re-saved successfully.

### 5b.1 Requester subs (summary)

| Sub | Behavior |
|---|---|
| `read` | Owned template → values: name, `type` via `toEnumForSelect(..., "messageTemplateType")`, `messageChannel` via `MessageChannel.forSelect(lang)` (or null), subject, body, variables, form field `meta_template` as `{ value, label }` rebuilt from `meta_template_id` + `meta_template_label` — **no** `messageTemplate` id key |
| `create` | Validate by kind → `createMessageTemplate` → `SUCCESS_CREATE` |
| `update` | Owned template + locked type → update attrs → `SUCCESS_UPDATE` |
| `delete` | Owned template; blocked if Meeting still links it (`CANNOT_DELETE_USED`) → else hard `destroy` → `SUCCESS_DELETE` |

Select hydrate pattern (enums / entity refs on `read`): [`../patterns/requester-read-select-hydrate.md`](../patterns/requester-read-select-hydrate.md).

## 6) Seed / ops

No template seed.

`body` is nullable in the Sequelize model. Existing Postgres tables created when body was required still have `NOT NULL`; `ADWHATS_PRO` create writes `body: null` and will fail until `ALTER TABLE message_templates ALTER COLUMN body DROP NOT NULL` (or a normal DatabaseConsole alter).

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — GQL mirrors + list/form UI (`flow-customer-message-templates.md`) |
| `cpanel/` | Deferred — no supervisor template surface |

Verification scripts (existing):

- Backend: `yarn generate-types`, `yarn type-check`
- Website (after mirror copy): `yarn type-check`

Generated codegen also refreshes `backend/src/app/gql/gql-types/supervisor.ts` (shared base enum types) even though supervisor has no template roots.

## 8) Failure modes

| Surface | Condition | Behavior |
|---|---|---|
| `messageTemplates` / `messageTemplate` | no `context.customer` | `NOT_PERMIT` |
| `messageTemplates` / `messageTemplate` | customer has no organization | `404` |
| `messageTemplate(id)` | missing / other-org id | framework empty → `404` |
| requester create/update | `ADWHATS_PRO` without `meta_template` / `variables` | Joi field errors |
| requester create | `ADWHATS_PRO` when DB `body` is still `NOT NULL` | Postgres `23502` until `ALTER … DROP NOT NULL` |
| requester delete | Meeting still links the template | `CANNOT_DELETE_USED` |
| `adwhatsProApprovedTemplates` | missing customer / other-org / non-Pro / empty token or account / remote error | `[]` |

## 9) Traceability map

| Path | Role | § |
|---|---|---|
| `backend/src/app/orm/models/MessageTemplate.ts` | nullable `body`; `meta_template_id` + `meta_template_label` | §3 |
| `backend/src/app/orchestrator/requesters/MessageTemplateRequester.ts` | form `meta_template`; split write; `body ?? null` | §5b |
| `backend/src/app/orm/models/Customer.ts` | `Ability.MESSAGE_TEMPLATE` | §5b |
| `backend/src/app/validation/joi_rules.ts` | `isCustomerOwnedMessageTemplate` | §5b |
| `backend/requesters.website.ts` | `customer.messageTemplate` | §5b |
| `backend/src/app/gql/definitions/base.graphql` | `_MessageTemplateType*` | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_MessageTemplate` + `adwhatsProApprovedTemplates` | §4 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | generated |
| `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts` | Thin bridge | §4 |
| `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | `me` → Organization | §4–§5 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | nest parents | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Pro template helper resolver | §4 |
| `backend/src/app/helpers/AdWhatsProDevApi.ts` | `listAdwhatsProApprovedTemplates` (`GET /templates`, no timeout) | §4 |
| `backend/src/app/helpers/CustomerAdwhatsLists.ts` | org-owned Pro channel gate | §4 |
| `backend/src/resources/trans/ar/general.ts` / `en/general.ts` | `messageTemplateType` | §3.5 |
| Website form, directory, pickers | portal UI | `flow-customer-message-templates.md` |

Channel credentials, account helper roots, and Pro used-channel lock: `message-channel-domain.md` §10.

## Related

- `docs/platforms/backend/contracts/message-channel-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/member-domain.md` (same org-owned GQL parent pattern)
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `.cursor/rules/message-template-domain.mdc`
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
- `.cursor/rules/organization-tenant-ownership.mdc`
