# Message Channel Domain Contract (Current)

## 1) Scope

Current Ejtmaa message-channel surface:

- ORM persistence for organization-owned **delivery channel** credentials (`message_channels`),
- channel types `CUSTOM_EMAIL` | `ADWHATS` | `ADWHATS_PRO`,
- status `ACTIVE` | `DISABLED` (default `ACTIVE`),
- customer GraphQL **read** of channels for the authenticated customer's organization,
- website GQL mirrors for that customer surface.

Out of scope (not shipped):

- channel requesters / write mutations / create-time connectivity test,
- send pipeline that flips `status` to `DISABLED` on token/connection failure,
- linking `MessageTemplate` rows to a channel (templates still use legacy `messageTemplateChannel` `WHATSAPP` | `EMAIL` — see `message-template-domain.md`),
- `MEETING_EMAIL` as a channel type (platform mailer templates intentionally have **no** channel row — product decision; not coded on templates yet),
- supervisor MessageChannel GraphQL,
- cpanel mirrors/UI,
- seed rows for channels,
- nested `_Organization.messageChannels` (use root list; same org-scoped root pattern as members/templates).

## 2) Domain purpose

`MessageChannel` is a **non-actor** delivery-account row inside an `Organization`.

- Holds **how** to send (SMTP or Ad Whats credentials), not message copy.
- `type` discriminates which credential columns apply.
- `status` gates usability: default `ACTIVE` (create flow is expected to connectivity-test before leaving active); future send failures / bad tokens may set `DISABLED`.
- Tenant boundary is `organization_id` (not `customer_id` directly).
- Distinct from legacy `_MessageTemplateChannel` enum (`WHATSAPP` | `EMAIL`) on `MessageTemplate`.

### Locked product decisions (templates — not yet implemented)

These constrain the next template refactor; they are **not** present on `MessageTemplate` today:

| Delivery kind | Channel row | Template content |
|---|---|---|
| Meeting email (platform mailer) | none (`message_channel_id` null) | `subject` + `body` required |
| Custom email | required `CUSTOM_EMAIL` channel | `subject` + `body` required |
| Ad Whats (unofficial) | required `ADWHATS` channel | `body` only |
| Ad Whats Pro (official) | required `ADWHATS_PRO` channel | provider variable map only (`{{key}}` → `{{n}}`); no free subject/body |

Enum spelling: `ADWHATS` / `ADWHATS_PRO` (not `AD_WHATS`).

## 3) ORM model

File: `backend/src/app/orm/models/MessageChannel.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `messageChannel`
- `tableName`: `message_channels`

### 3.1 Attrs layout

- `//relations` — `organization_id`
- `//info` — `name`, `type`, `status`, SMTP fields, Ad Whats fields

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `organization_id` | BIGINT | no | FK → `organizations.id`, real constraint |
| `name` | STRING(191) | no | picker / library label |
| `type` | STRING(191) | no | enum metadata `messageChannelType`; typed object |
| `status` | STRING(191) | no | enum metadata `messageChannelStatus`; default `ACTIVE`; typed object |
| `smtp_host` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `smtp_port` | INTEGER | yes | `CUSTOM_EMAIL` |
| `smtp_secure` | BOOLEAN | yes | `CUSTOM_EMAIL` |
| `smtp_user` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `smtp_password` | TEXT | yes | `CUSTOM_EMAIL`; secret |
| `from_name` | STRING(191) | yes | `CUSTOM_EMAIL` optional |
| `from_address` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `adwhats_token` | TEXT | yes | `ADWHATS` / `ADWHATS_PRO`; secret |
| `adwhats_account_id` | STRING(191) | yes | `ADWHATS` / `ADWHATS_PRO` |

Exported TS types:

- `MessageChannelType = keyof G_Tr["enums"]["messageChannelType"]`
- `MessageChannelStatus = keyof G_Tr["enums"]["messageChannelStatus"]`

ORM does **not** enforce per-type nullability (all credential columns nullable). Write-path Joi/requester must require/forbid columns by `type` when shipped.

### 3.3 Indexes

- `message_channels_organization_id`
- `message_channels_organization_id_type`
- `message_channels_organization_id_status`

### 3.4 Relations

`MessageChannel.boot()`:

- `belongsTo(Organization)` on `organization_id` (real FK)

`Organization.boot()` inverse:

- `hasMany(MessageChannel)` on `organization_id` (real FK)
- mixins: `getMessageChannels` / `createMessageChannel` / … (association PK type `number`)

No `hasMany(MessageTemplate)` yet (template FK not shipped).

### 3.5 Localization

Enum keys under:

- `backend/src/resources/trans/ar/general.ts`
- `backend/src/resources/trans/en/general.ts`

`messageChannelType`: `CUSTOM_EMAIL`, `ADWHATS`, `ADWHATS_PRO`.

`messageChannelStatus`: `ACTIVE`, `DISABLED`.

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MessageChannelTypeValue` / `_MessageChannelType`, `_MessageChannelStatusValue` / `_MessageChannelStatus`
- `backend/src/app/gql/definitions/customer.graphql` — `_MessageChannel` + roots

### Type `_MessageChannel`

Implements `_Timestamps` & `_Pagination`.

Exposed fields:

- info: `id`, `name`, `type`, `status`
- `CUSTOM_EMAIL` non-secrets: `smtp_host`, `smtp_port`, `smtp_secure`, `smtp_user`, `from_name`, `from_address`
- Ad Whats non-secret: `adwhats_account_id`
- timestamps, `organization`, `total_count`

**Not in SDL (secrets):** `smtp_password`, `adwhats_token`.

### Root queries

- `messageChannels: [_MessageChannel]`
- `messageChannel(id: ID!): _MessageChannel`

Resolvers (`CustomerSchema`):

```ts
prepareManyGQLModels({ me: true })
prepareOneGQLModel({ me: true, id })
```

### Bridge: `MessageChannelBridge`

File: `backend/src/app/gql/bridges/customer/MessageChannelBridge.ts`

- Extends `CustomerOrganizationOwnedBridgeBase`
- `ident = "messageChannel"`, `typeIdent = "_MessageChannel"`, `ormModel = MessageChannelModel`
- `GetManyParent = OrganizationOwnedMeParent` (`{ me: true }`)
- `GetOneParent = MessageChannelModel | { me: true; id: string }`

### Inverse relation parent typing

`OrganizationBridge.GetOneParent` includes `MessageChannelModel` so `_MessageChannel.organization` resolves.

### Registered bridges

`CustomerSchema.registeredBridges` includes `MessageChannelBridge`.

## 5) Read flow (root)

### `messageChannels`

1. `MessageChannelBridge.AsRoot` → `prepareManyGQLModels({ me: true })`
2. `CustomerOrganizationOwnedBridgeBase.getRootOrmParent` → customer's Organization
3. Base list options: `withListable` + `withReplacements` + `order updated_at DESC`
4. Association load via Organization → message channels

### `messageChannel(id)`

1. `prepareOneGQLModel({ me: true, id })`
2. Same org resolve as above
3. Base one options: `where: { id }`
4. Scoped to that organization association

## 6) Seed

No channel seed in this change set.

## 7) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — `base.graphql`, `customer.graphql`, `gql-types/base.ts`, `gql-types/customer.ts` synced |
| `cpanel/` | Deferred — no supervisor channel surface |

Backend verification: `yarn generate-types`, `yarn type-check`.

Local registry (gitignored): `backend/.types/models.ts` must include `"MessageChannel"`.

## 8) Failure modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| `messageChannels` / `messageChannel` | no `context.customer` | `NOT_PERMIT` |
| `messageChannels` / `messageChannel` | customer has no organization | `404` |
| `messageChannel(id)` | missing / other-org id | framework empty → `404` |

## 9) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/MessageChannel.ts` | ORM source of truth | §3 |
| `backend/src/app/orm/models/Organization.ts` | `hasMany MessageChannel` + mixins | §3.4 |
| `backend/src/resources/trans/ar/general.ts` | `messageChannelType` / `messageChannelStatus` AR | §3.5 |
| `backend/src/resources/trans/en/general.ts` | EN mirrors | §3.5 |
| `backend/src/app/gql/definitions/base.graphql` | type/status GQL enums | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_MessageChannel` + roots | §4 |
| `backend/src/app/gql/bridges/customer/MessageChannelBridge.ts` | thin entity bridge | §4–§5 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `MessageChannelModel` | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register + resolvers | §4 |
| `backend/src/app/gql/gql-types/base.ts` | Generated (includes channel enums) | §7 — generated |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §7 — generated |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated (base enums pulled in) | §7 — generated; no supervisor channel roots |
| `website/src/types/gql/definitions/base.graphql` | Mirror | §7 — generated/mirror |
| `website/src/types/gql/definitions/customer.graphql` | Mirror | §7 — generated/mirror |
| `website/src/types/gql/gql-types/base.ts` | Mirror | §7 — generated/mirror |
| `website/src/types/gql/gql-types/customer.ts` | Mirror | §7 — generated/mirror |
| `backend/.types/models.ts` | Local registry key `MessageChannel` | §7 — gitignored |
| `.cursor/rules/organization-tenant-ownership.mdc` | Tenant + GQL ownership notes | governance |
| `.cursor/rules/message-channel-domain.mdc` | Channel invariants | governance |
| `.cursor/skills/orm-model-generator/SKILL.md` | Model addendum | governance |

## Related

- `docs/platforms/backend/contracts/message-template-domain.md` (legacy template channel enum; not yet linked to `MessageChannel`)
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md` (optional template FKs on Meeting)
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/patterns/gql-role-bridge-base-contract.md`
- `.cursor/rules/organization-tenant-ownership.mdc`
- `.cursor/rules/message-channel-domain.mdc`
