# Message Template Domain Contract (Current)

## 1) Scope

Current Ejtmaa message-template surface:

- ORM persistence for organization-owned notification message templates,
- channel types WhatsApp and Email,
- customer GraphQL read of templates for the authenticated customer's organization,
- website GQL mirrors for that customer surface.

Out of scope (not shipped):

- template requesters / write mutations,
- supervisor MessageTemplate GraphQL,
- cpanel mirrors/UI (`cpanel/` checkout temporarily absent),
- seed rows for templates,
- nested `_Organization.messageTemplates` (use root list; same org-scoped root pattern as members).

Related shipped consumer: `Meeting` optional FKs `whatsapp_template_id` / `email_template_id` — see `meeting-domain.md`.

## 2) Domain purpose

`MessageTemplate` is a **non-actor** reusable message library row inside an `Organization`.

- Channel is `WHATSAPP` or `EMAIL` (localized enum `messageTemplateChannel`).
- Email may carry `subject`; WhatsApp uses `body` only (`subject` null).
- Tenant boundary is `organization_id` (not `customer_id` directly).
- Templates are referenced by meetings via optional FKs; meetings do not store inline template text columns.

## 3) ORM model

File: `backend/src/app/orm/models/MessageTemplate.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `messageTemplate`
- `tableName`: `message_templates`

### 3.1 Attrs layout

- `//relations` — `organization_id`
- `//info` — `name`, `channel`, `subject`, `body`

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `organization_id` | BIGINT | no | FK → `organizations.id`, real constraint |
| `name` | STRING(191) | no | library label |
| `channel` | STRING(191) | no | enum metadata `messageTemplateChannel`; typed object |
| `subject` | STRING(191) | yes | email subject; null for WhatsApp |
| `body` | TEXT | no | message body |

Exported TS type: `MessageTemplateChannel = keyof G_Tr["enums"]["messageTemplateChannel"]`.

### 3.3 Indexes

- `message_templates_organization_id` — list by org
- `message_templates_organization_id_channel` — list/filter by org + channel

### 3.4 Relations

`MessageTemplate.boot()`:

- `belongsTo(Organization)` on `organization_id` (real FK)

`Organization.boot()` inverse:

- `hasMany(MessageTemplate)` on `organization_id` (real FK)
- mixins: `getMessageTemplates` / `createMessageTemplate` / … (association PK type `number`)

No direct `Customer` ↔ `MessageTemplate` association.

### 3.5 Localization

Enum key `messageTemplateChannel` under:

- `backend/src/resources/trans/ar/general.ts`
- `backend/src/resources/trans/en/general.ts`

Values: `WHATSAPP`, `EMAIL`.

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MessageTemplateChannelValue` / `_MessageTemplateChannel`
- `backend/src/app/gql/definitions/customer.graphql` — `_MessageTemplate` + roots

### Type `_MessageTemplate`

Implements `_Timestamps` & `_Pagination`.

Fields: `id`, `name`, `channel`, `subject`, `body`, timestamps, `organization`, `total_count`.

### Root queries

- `messageTemplates: [_MessageTemplate]`
- `messageTemplate(id: ID!): _MessageTemplate`

Resolvers (`CustomerSchema`):

```ts
prepareManyGQLModels({ me: true })
prepareOneGQLModel({ me: true, id })
```

### Bridge: `MessageTemplateBridge`

File: `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts`

- Extends `CustomerOrganizationOwnedBridgeBase` (not bare `CustomerBridgeBase`)
- `ident = "messageTemplate"`, `typeIdent = "_MessageTemplate"`, `ormModel = MessageTemplateModel`
- `GetManyParent = OrganizationOwnedMeParent` (`{ me: true }`)
- `GetOneParent = MessageTemplateModel | MeetingModel | { me: true; id: string }` (`MeetingModel` for `_Meeting` template nests)
- Does **not** override `getRootOrmParent` or `getOrmFindOptions` (inherited / role defaults)

### Shared base: `CustomerOrganizationOwnedBridgeBase`

File: `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts`

For org-owned customer children (`Member`, `MessageTemplate`, …):

- On `{ me: true }`: require `context.customer`, load `getOrganization()`, return Organization or throw `NOT_PERMIT` / `404`
- Otherwise delegate to `CustomerBridgeBase.getRootOrmParent`

Do **not** copy this resolve into each entity bridge. Do **not** put this logic on `CustomerBridgeBase` (other customer bridges resolve differently).

### Inverse relation parent typing (mandatory)

When `_MessageTemplate.organization` is selected, the framework prepares **`OrganizationBridge`** with parent = `MessageTemplateModel`.

Therefore `OrganizationBridge` must declare:

```ts
export type GetOneParent =
    | CustomerModel
    | MemberModel
    | MessageTemplateModel
    | MeetingModel;
```

`MessageTemplateBridge.GetOneParent` also includes `MeetingModel` for nested `_Meeting.whatsappTemplate` / `emailTemplate`.

### Registered bridges

`CustomerSchema.registeredBridges` includes `MessageTemplateBridge`.

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

## 6) Seed

No template seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — `base.graphql`, `customer.graphql`, `gql-types/base.ts`, `gql-types/customer.ts` synced |
| `cpanel/` | Deferred — platform checkout absent; no supervisor template surface |

Backend verification: `yarn generate-types`, `yarn type-check`.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `messageTemplates` / `messageTemplate` | no `context.customer` | `NOT_PERMIT` |
| `messageTemplates` / `messageTemplate` | customer has no organization | `404` |
| `messageTemplate(id)` | missing / other-org id | framework empty → `404` |

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/MessageTemplate.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Organization.ts` | `hasMany MessageTemplate` + mixins | §3.4 |
| `backend/src/resources/trans/ar/general.ts` | `messageTemplateChannel` AR | §3.5 |
| `backend/src/resources/trans/en/general.ts` | `messageTemplateChannel` EN | §3.5 |
| `backend/src/app/gql/definitions/base.graphql` | channel GQL enum wrapper | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_MessageTemplate` + roots + inverse | §4 |
| `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` | shared `me` → Organization | §4 |
| `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts` | thin entity bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | same org-owned base (refactor) | §4 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `MessageTemplateModel` | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register + resolvers | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated (includes channel types) | §7 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enum types pulled in) | §7 — no supervisor template roots |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 |
| `backend/.types/models.ts` | Generated registry (gitignored) | excluded from narrative — regenerated on ORM boot / local tooling |

## Related

- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/member-domain.md` (same org-owned GQL parent pattern)
- `docs/platforms/backend/contracts/meeting-domain.md` (optional template FKs from Meeting)
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `docs/invariants/backend.md` (B15, B18)
- `.cursor/rules/gql-root-parent-payload-contract.mdc`
- `.cursor/rules/organization-tenant-ownership.mdc`
