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

Out of scope (not shipped in this slice):

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
| `ADWHATS_PRO` | required, channel `type=ADWHATS_PRO` | `meta_template_id` + `variables` map (no free subject/body) |

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
- `ADWHATS_PRO`: required `meta_template_id` + `FormMessageTemplateVariablesField` — human labels only (no `{{token}}` in UI); default variable numbers `1`–`4`; editable to match approved WhatsApp template order.

**Retired (do not reintroduce):** column/enum `channel` / `messageTemplateChannel` / values `WHATSAPP` | `EMAIL`; GQL `_MessageTemplateChannel*`.

## 3) ORM model

File: `backend/src/app/orm/models/MessageTemplate.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `messageTemplate`
- `tableName`: `message_templates`

### 3.1 Attrs layout

- `//relations` — `organization_id`, `message_channel_id`
- `//info` — `name`, `type`, `subject`, `body`, `variables`, `meta_template_id`

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
| `meta_template_id` | STRING(191) | yes | Ad Whats Pro only; approved Meta template identifier; write-required when `type=ADWHATS_PRO` |

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

**No** `hasMany(MessageTemplate)` on `MessageChannel` in this slice:

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
| `EJTMAA_EMAIL` | Ejtmaa email | بريد اجتماع |
| `CUSTOM_EMAIL` | Custom email | بريد مخصص |
| `ADWHATS` | Ad Whats | أد واتس |
| `ADWHATS_PRO` | Ad Whats Pro | أد واتس برو |

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MessageTemplateTypeValue` / `_MessageTemplateType`
- `backend/src/app/gql/definitions/customer.graphql` — `_MessageTemplate` + roots

### Type `_MessageTemplate`

Implements `_Timestamps` & `_Pagination`.

Fields: `id`, `name`, `type`, `subject`, `body`, `variables` (`JSONObject`), `meta_template_id`, timestamps, `organization`, `messageChannel`, `total_count`.

### Root queries

- `messageTemplates: [_MessageTemplate]`
- `messageTemplate(id: ID!): _MessageTemplate`

Resolvers (`CustomerSchema`): `prepareManyGQLModels({ me: true })` / `prepareOneGQLModel({ me: true, id })`.

`CustomerSchema.registeredBridges` includes `MessageTemplateBridge`.

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

`MessageChannelBridge.GetOneParent` therefore includes `MessageTemplateModel` (shipped in this change set).

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
- Website map: `customer.messageTemplate` in `requesters.website.ts` (mirrored on website — W18).
- **`read` values must not echo `messageTemplate` id** — identity stays in form `initProps.values`.
- Update **locks** `type` to the existing row (client must echo the read value).
- Channel FK: null for `EJTMAA_EMAIL`; otherwise org-owned `MessageChannel` whose `type` matches template `type`.
- Content columns required by `type` via Joi `when`; unused branches `joi.any().optional().allow(null, "").strip()`. Persist with one flat write (`props.field ?? null`) — **no** per-type `attrsForType` helper. See `.cursor/rules/requester-type-conditional-strip.mdc`.
- Do **not** use `forbidden()` for leftover form content fields unless the client guarantees the key is absent; prefer `.strip()`.
- Channel write for credentials uses the same strip + `?? null` pattern (`MessageChannelRequester`).
- Template picker list filter: website always requests `status: ACTIVE` (connectivity stub may leave many rows `DISABLED`).

### 5b.1 Requester subs (summary)

| Sub | Behavior |
|---|---|
| `read` | Owned template → values: name, `type` via `toEnumForSelect(..., "messageTemplateType")`, `messageChannel` via `MessageChannel.forSelect(lang)` (or null), subject, body, variables, `meta_template_id` — **no** `messageTemplate` id key |
| `create` | Validate by kind → `createMessageTemplate` → `SUCCESS_CREATE` |
| `update` | Owned template + locked type → update attrs → `SUCCESS_UPDATE` |
| `delete` | Owned template; blocked if Meeting still links it (`CANNOT_DELETE_USED`) → else hard `destroy` → `SUCCESS_DELETE` |

Select hydrate pattern (enums / entity refs on `read`): [`../patterns/requester-read-select-hydrate.md`](../patterns/requester-read-select-hydrate.md).

## 6) Seed / migration note

No template seed in this change set.

Applying the new columns against DBs that still have legacy `channel` / `WHATSAPP`|`EMAIL` requires schema sync and a data migration — **not** automated here. Ops must plan before promote.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — GQL mirrors + list/form UI (`flow-customer-message-templates.md`) |
| `cpanel/` | Deferred — no supervisor template surface |

Verification scripts (existing):

- Backend: `yarn generate-types`, `yarn type-check`
- Website (after mirror copy): `yarn type-check`

Generated codegen also refreshes `backend/src/app/gql/gql-types/supervisor.ts` (shared base enum types) even though supervisor has no template roots.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `messageTemplates` / `messageTemplate` | no `context.customer` | `NOT_PERMIT` |
| `messageTemplates` / `messageTemplate` | customer has no organization | `404` |
| `messageTemplate(id)` | missing / other-org id | framework empty → `404` |

## 9) Traceability map (change-set inventory — this slice)

### Backend (`backend/` repo)

| Path | Status | Role | § |
|---|---|---|---|
| `src/app/orm/models/MessageTemplate.ts` | M | `meta_template_id` + content attrs | §3 |
| `src/app/orm/models/MessageChannel.ts` | M | `forSelect(lang)` | hydrate |
| `src/app/orm/models/Customer.ts` | M | `Ability.MESSAGE_TEMPLATE` + `can` | §5b |
| `src/app/validation/joi_rules.ts` | M | `isCustomerOwnedMessageTemplate` | §5b |
| `src/app/orchestrator/requesters/MessageTemplateRequester.ts` | A | CRUD; `.strip()` unused; write `?? null` | §5b |
| `src/app/orchestrator/requesters/MessageChannelRequester.ts` | M | same strip write pattern for credentials | channel domain |
| `requesters.website.ts` | M | `customer.messageTemplate` | §5b |
| `src/app/gql/bridges/customer/MessageChannelBridge.ts` | M | `filter.type` + `filter.status`; `GetOneParent` += template | §4 / channel |
| `src/app/gql/schemas/CustomerSchema.ts` | M | pass `filter` into `messageChannels` | channel |
| `src/app/gql/definitions/customer.graphql` | M | `meta_template_id`; `_MessageChannelFilter.status` | §4 |
| `src/app/gql/gql-types/customer.ts` | M | Generated | §7 |
| `src/resources/trans/ar/general.ts` | M | enum labels as needed | §3.5 |

### Backend — supporting (may pre-exist; required)

| Path | Role | § |
|---|---|---|
| `src/app/gql/bridges/customer/MessageTemplateBridge.ts` | Thin bridge | §4 |
| `src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | `me` → Organization | §4–§5 |
| `src/app/gql/bridges/customer/OrganizationBridge.ts` | nest parents | §4 |
| `src/app/gql/definitions/base.graphql` | `_MessageTemplateType*` | §4 |
| `src/resources/trans/en/general.ts` | EN enum mirror | §3.5 |

### Website — see `docs/platforms/website/flow-customer-message-templates.md` §7

### Workspace root — docs / governance

| Path | Role |
|---|---|
| `docs/platforms/backend/contracts/message-template-domain.md` | This contract |
| `docs/platforms/backend/contracts/message-channel-domain.md` | Filter + strip credentials |
| `docs/platforms/backend/patterns/requester-read-select-hydrate.md` | Select hydrate |
| `docs/platforms/website/flow-customer-message-templates.md` | Website flow + inventory |
| `docs/platforms/website/flow-customer-message-channels.md` | Channel UI sibling |
| `docs/platforms/website/flow-customer-shell.md` | Drawer shipped list |
| `docs/platforms/website/flow-form-foundation.md` | Wrapper / picker |
| `docs/platforms/website/route-registry-contract.md` | Routes index |
| `docs/platforms/website/README.md` | Flow index |
| `docs/platforms/backend/README.md` | Contracts index |
| `.cursor/rules/message-template-domain.mdc` | Template invariants |
| `.cursor/rules/message-channel-domain.mdc` | Channel invariants |
| `.cursor/rules/requester-read-select-hydrate.mdc` | Read SelectOption |
| `.cursor/rules/requester-type-conditional-strip.mdc` | Joi unused `.strip()` |
| `.cursor/rules/website-message-template-form-ux.mdc` | Website form UX locks |
| `.cursor/skills/website-customer-message-templates/SKILL.md` | Website workflow |
| `.cursor/skills/website-entity-picker/SKILL.md` | Picker workflow |
| `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md` | Hydrate workflow |
| `backend` / `website` | Nested git pointers (submodule-style) |

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
