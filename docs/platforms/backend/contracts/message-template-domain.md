# Message Template Domain Contract (Current)

## 1) Scope

Current Ejtmaa message-template surface:

- ORM persistence for organization-owned reusable notification templates,
- delivery kinds `EJTMAA_EMAIL` | `CUSTOM_EMAIL` | `ADWHATS` | `ADWHATS_PRO`,
- optional FK to `MessageChannel` (`message_channel_id` null only for `EJTMAA_EMAIL`),
- content columns by kind (`subject` / `body` / `variables`),
- customer GraphQL **read** of templates for the authenticated customer's organization,
- website GQL mirrors for that customer surface (`base` + `customer` SDL + gql-types).

Out of scope (not shipped in this slice):

- template requesters / write mutations / `Customer.Ability.MESSAGE_TEMPLATE`,
- supervisor MessageTemplate GraphQL,
- cpanel mirrors/UI,
- seed rows for templates,
- nested `_Organization.messageTemplates` (use root list),
- inverse `MessageChannel.hasMany(MessageTemplate)` (template `belongsTo` only; see §3.4),
- renaming Meeting FKs `whatsapp_template_id` / `email_template_id` (still point at `MessageTemplate` rows; column names are legacy).

Related: `message-channel-domain.md` (delivery credentials). Meeting optional template FKs: `meeting-domain.md`.

## 2) Domain purpose

`MessageTemplate` is a **non-actor** reusable message library row inside an `Organization`.

| `type` | `message_channel_id` | Content (write-path product rules; not ORM-enforced) |
|---|---|---|
| `EJTMAA_EMAIL` | null (platform mailer) | `subject` + `body` required on write |
| `CUSTOM_EMAIL` | required, channel `type=CUSTOM_EMAIL` | `subject` + `body` required on write |
| `ADWHATS` | required, channel `type=ADWHATS` | `body` only (`subject` null) |
| `ADWHATS_PRO` | required, channel `type=ADWHATS_PRO` | `variables` map only (no free subject/body) |

- Tenant boundary is `organization_id` (not `customer_id` directly).
- Templates are referenced by meetings via optional FKs; meetings do not store inline template text.
- ORM does **not** enforce per-type nullability; write-path Joi/requester (todo) must.
- Credentials never live on the template row — only on `MessageChannel` (or the platform mailer for `EJTMAA_EMAIL`).

`variables` shape (Ad Whats Pro): `Record<string, string>` — template placeholder key → provider slot (e.g. `name` → `1` meaning `{{name}}` → `{{1}}`). Stored as JSONB (`isTypedObject: true`).

**Retired (do not reintroduce):** column/enum `channel` / `messageTemplateChannel` / values `WHATSAPP` | `EMAIL`; GQL `_MessageTemplateChannel*`.

## 3) ORM model

File: `backend/src/app/orm/models/MessageTemplate.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `messageTemplate`
- `tableName`: `message_templates`

### 3.1 Attrs layout

- `//relations` — `organization_id`, `message_channel_id`
- `//info` — `name`, `type`, `subject`, `body`, `variables`

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

Fields: `id`, `name`, `type`, `subject`, `body`, `variables` (`JSONObject`), timestamps, `organization`, `messageChannel`, `total_count`.

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

## 6) Seed / migration note

No template seed in this change set.

Applying the new columns against DBs that still have legacy `channel` / `WHATSAPP`|`EMAIL` requires schema sync and a data migration — **not** automated here. Ops must plan before promote.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — mirrored `base.graphql`, `customer.graphql`, `gql-types/base.ts`, `gql-types/customer.ts` |
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

## 9) Traceability map (change-set inventory)

### Backend (`backend/` repo) — modified

| Path | Role | Section |
|---|---|---|
| `src/app/orm/models/MessageTemplate.ts` | ORM attrs/indexes/relations | §3 |
| `src/app/orm/models/Organization.ts` | `hasMany` mixin PK `string` (templates + channels) | §3.4 |
| `src/app/gql/bridges/customer/MessageChannelBridge.ts` | `GetOneParent` += `MessageTemplateModel` | §4 |
| `src/app/gql/definitions/base.graphql` | `_MessageTemplateType*` | §4 |
| `src/app/gql/definitions/customer.graphql` | `_MessageTemplate` fields | §4 |
| `src/resources/trans/ar/general.ts` | `messageTemplateType` AR | §3.5 |
| `src/resources/trans/en/general.ts` | `messageTemplateType` EN | §3.5 |
| `src/app/gql/gql-types/base.ts` | Generated | §7 |
| `src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `src/app/gql/gql-types/supervisor.ts` | Generated (base enums) | §7 |

### Backend — unchanged but required for the surface

| Path | Role | Section |
|---|---|---|
| `src/app/gql/bridges/customer/MessageTemplateBridge.ts` | Thin entity bridge | §4 |
| `src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | `me` → Organization | §4–§5 |
| `src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `MessageTemplateModel` | §4 |
| `src/app/gql/schemas/CustomerSchema.ts` | Register + root resolvers | §4–§5 |
| `src/app/orm/models/MessageChannel.ts` | Credential peer; no inverse `hasMany` | §3.4 |

### Website (`website/` repo) — modified

| Path | Role | Section |
|---|---|---|
| `src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `src/types/gql/gql-types/base.ts` | Generated mirror | §7 |
| `src/types/gql/gql-types/customer.ts` | Generated mirror | §7 |
| `lib/tsconfig.tsbuildinfo` | Build cache noise | excluded from narrative |

### Workspace root — docs / governance

| Path | Role | Section |
|---|---|---|
| `docs/platforms/backend/contracts/message-template-domain.md` | This contract | — |
| `docs/platforms/backend/contracts/message-channel-domain.md` | Cross-ref + locked content table | related |
| `docs/platforms/backend/contracts/graphql-and-types.md` | Index: enum + nest | related |
| `docs/platforms/backend/overview.md` | Model one-liner | related |
| `docs/platforms/backend/README.md` | Contracts index blurb | related |
| `.cursor/rules/message-template-domain.mdc` | Agent invariants | governance |
| `.cursor/rules/message-channel-domain.mdc` | No `EJTMAA_EMAIL` channel type | governance |
| `.cursor/rules/organization-tenant-ownership.mdc` | Org ownership bullets | governance |
| `.cursor/skills/orm-model-generator/SKILL.md` | Model addendum | governance |

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
