# Message Channel Domain Contract (Current)

## 1) Scope

Current Ejtmaa message-channel surface:

- ORM persistence for organization-owned **delivery channel** credentials (`message_channels`),
- channel types `CUSTOM_EMAIL` | `ADWHATS` | `ADWHATS_PRO`,
- status `ACTIVE` | `DISABLED` (default `ACTIVE`),
- customer GraphQL **read** of channels for the authenticated customer's organization,
- website GQL mirrors for that customer surface.

Out of scope (not shipped):

- Real SMTP / Ad Whats connectivity inside `testConnection()` (method exists; stub returns `false`; create/update **do** call it and set `ACTIVE` / `DISABLED` from the result),
- send pipeline that flips `status` to `DISABLED` on token/connection failure after send,
- linking `MessageTemplate` content rules to requesters (shipped — see `message-template-domain.md` §5b),
- `EJTMAA_EMAIL` as a channel type (platform mailer templates intentionally have **no** channel row — product decision),
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
- Aligns with `MessageTemplate.type` delivery kinds (`message-template-domain.md`); credentials stay on `MessageChannel` only.

### Locked product decisions (templates)

Authoritative on `MessageTemplate` now — see `message-template-domain.md`:

| Template `type` | Channel row | Template content |
|---|---|---|
| `EJTMAA_EMAIL` (platform mailer) | none (`message_channel_id` null) | `subject` + `body` required |
| `CUSTOM_EMAIL` | required matching channel | `subject` + `body` required |
| `ADWHATS` | required matching channel | `body` only |
| `ADWHATS_PRO` | required matching channel | `meta_template_id` + provider variable map (`{{key}}` → `{{n}}`); no free subject/body |

Enum spelling: `ADWHATS` / `ADWHATS_PRO` (not `AD_WHATS`). Write-path enforcement is requester todo.

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
| `id` | BIGINT PK | no | auto-increment; Attrs typed `string` like Member/Meeting |
| `organization_id` | BIGINT | no | FK → `organizations.id`, real constraint |
| `name` | STRING(191) | no | picker / library label |
| `type` | STRING(191) | no | enum metadata `messageChannelType`; typed object |
| `status` | STRING(191) | no | enum metadata `messageChannelStatus`; default `ACTIVE`; typed object |
| `smtp_host` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `smtp_port` | INTEGER | yes | `CUSTOM_EMAIL` |
| `smtp_secure` | BOOLEAN | yes | `CUSTOM_EMAIL` |
| `smtp_user` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `smtp_password` | TEXT | yes | `CUSTOM_EMAIL` |
| `from_name` | STRING(191) | yes | `CUSTOM_EMAIL` optional |
| `from_address` | STRING(191) | yes | `CUSTOM_EMAIL` |
| `adwhats_token` | TEXT | yes | `ADWHATS` / `ADWHATS_PRO` |
| `adwhats_account_id` | STRING(191) | yes | `ADWHATS` / `ADWHATS_PRO` |

Exported TS types:

- `MessageChannelType = keyof G_Tr["enums"]["messageChannelType"]`
- `MessageChannelStatus = keyof G_Tr["enums"]["messageChannelStatus"]`

ORM does **not** enforce per-type nullability (all credential columns nullable). Write-path Joi/requester requires/forbids columns by `type`.

### 3.3 Indexes

- `message_channels_organization_id`
- `message_channels_organization_id_type`
- `message_channels_organization_id_status`

### 3.4 Relations

`MessageChannel.boot()`:

- `belongsTo(Organization)` on `organization_id` (real FK)

`Organization.boot()` inverse:

- `hasMany(MessageChannel)` on `organization_id` (real FK)
- mixins: `getMessageChannels` / `createMessageChannel` / … (association PK type `string`, like Member)

No `hasMany(MessageTemplate)` on `MessageChannel` in this slice (template `belongsTo` channel only; avoids circular import).

### 3.5 Localization

Enum keys under:

- `backend/src/resources/trans/ar/general.ts`
- `backend/src/resources/trans/en/general.ts`

`messageChannelType`: `CUSTOM_EMAIL`, `ADWHATS`, `ADWHATS_PRO`.

`messageChannelStatus`: `ACTIVE`, `DISABLED`.

### 3.6 Connectivity stub

`MessageChannelModel.testConnection(): Promise<boolean>` — stub returns `false` until SMTP / Ad Whats clients are implemented. Create/update requesters call it and set `ACTIVE` / `DISABLED` from the result.

## 4) Customer GraphQL surface

SDL:

- `backend/src/app/gql/definitions/base.graphql` — `_MessageChannelTypeValue` / `_MessageChannelType`, `_MessageChannelStatusValue` / `_MessageChannelStatus`
- `backend/src/app/gql/definitions/customer.graphql` — `_MessageChannel` + `_MessageChannelFilter` + roots

### Filter `_MessageChannelFilter`

| Field | Type | Behavior |
|---|---|---|
| `type` | `_MessageChannelTypeValue` | Exact match on channel `type` (org-scoped list still via association) |
| `status` | `_MessageChannelStatusValue` | Exact match on channel `status` (e.g. template picker uses `ACTIVE` only) |

Root: `messageChannels(filter: _MessageChannelFilter)`. Singular `messageChannel(id)` ignores filter.

Bridge: `MessageChannelBridge.getOrmFindOptions` maps `filter.type` / `filter.status` when root-many (same pattern as `_MemberFilter` / `_MeetingFilter`).

### Type `_MessageChannel`

Implements `_Timestamps` & `_Pagination`.

Exposed fields:

- info: `id`, `name`, `type`, `status`
- `CUSTOM_EMAIL`: `smtp_host`, `smtp_port`, `smtp_secure`, `smtp_user`, `smtp_password`, `from_name`, `from_address`
- Ad Whats: `adwhats_token`, `adwhats_account_id`
- timestamps, `organization`, `total_count`

### Root queries

- `messageChannels(filter: _MessageChannelFilter): [_MessageChannel]`
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
- `GetOneParent = MessageChannelModel | MessageTemplateModel | { me: true; id: string }`
  - `MessageTemplateModel` for nested `_MessageTemplate.messageChannel`

### Inverse relation parent typing

`OrganizationBridge.GetOneParent` includes `MessageChannelModel` so `_MessageChannel.organization` resolves.

`MessageChannelBridge.GetOneParent` includes `MessageTemplateModel` so `_MessageTemplate.messageChannel` resolves (see `message-template-domain.md`).

### Registered bridges

`CustomerSchema.registeredBridges` includes `MessageChannelBridge`.

## 5) Customer write path

- Ability on `Customer`: `MESSAGE_CHANNEL` with `sub: "create" | "read" | "update" | "delete"` and optional `messageChannel` target.
  - `create`: customer must have an organization (`ACTION_NOT_ALLOWED` otherwise).
  - `read` / `update` / `delete`: resolve channel; must belong to that organization (`404` missing, `NOT_PERMIT` other org).
- Joi helper: `isCustomerOwnedMessageChannel` in `joi_rules.ts` (mirrors `isCustomerOwnedMember` shape).
- Requester: `MessageChannelRequester` (`@requester("messageChannel")`) — `read` | `create` | `update` | `delete` for website/customer.
- Website map: `customer.messageChannel` in `requesters.website.ts` (mirrored on website — W18).
- **`read` values must not echo `messageChannel` id** — identity stays in form `initProps.values`; form merge preserves it (same rule as `MemberRequester.read`).
- Update locks `type` to the existing row (client must echo the read value).
- Credential columns required by `type` via Joi `when`; unused branches use `joi.any().optional().allow(null, "").strip()`. Write path persists stripped props with `?? null` (no per-type attrs helper).
- `status` is **not** client-owned: create/update call `testConnection()` after credentials are written — `true` → `ACTIVE`, `false` → `DISABLED`.

### 5.1 Requester subs (summary)

| Sub | Behavior |
|---|---|
| `read` | Owned channel → values: name, `type` via `toEnumForSelect(..., "messageChannelType")`, credential columns (`smtp_secure` as `"true"`/`"false"` string for form choice) — **no** `messageChannel` id key |
| `create` | Validate → `createMessageChannel` with `status: ACTIVE` → `testConnection` may flip to `DISABLED` → `SUCCESS_CREATE` |
| `update` | Owned channel + locked type → update attrs → `testConnection` → `ACTIVE`/`DISABLED` → `SUCCESS_UPDATE` |
| `delete` | Owned channel → `destroy({ force: true })` → `SUCCESS_DELETE` |

Select hydrate pattern (enums / entity refs on `read`): [`../patterns/requester-read-select-hydrate.md`](../patterns/requester-read-select-hydrate.md).

## 6) Read flow (root)

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

## 7) Seed

No channel seed in this change set.

## 8) Frontend mirrors

| Platform | Status |
|---|---|
| `website/` | Active — SDL + gql-types synced; form + directory shipped |
| `cpanel/` | Deferred — no supervisor channel surface |

Backend verification: `yarn generate-types`, `yarn type-check`.

Local registry (gitignored): `backend/.types/models.ts` must include `"MessageChannel"`.

## 9) Failure modes

| Surface | Condition | Behavior |
|---|---|---|
| `messageChannels` / `messageChannel` | no `context.customer` | `NOT_PERMIT` |
| `messageChannels` / `messageChannel` | customer has no organization | `404` |
| `messageChannel(id)` | missing / other-org id | framework empty → `404` |
| requester update | client `type` ≠ locked row type | `NOT_PERMIT` |
| requester create/update | wrong-type / missing credential fields | Joi field errors (`when` + optional unused branches) |
| requester create/update | `testConnection()` false (current stub) | row saved as `DISABLED` |

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/MessageChannel.ts` | ORM + `testConnection` stub + `forSelect(lang)` | §3 / hydrate |
| `backend/src/app/orm/models/Customer.ts` | `Ability.MESSAGE_CHANNEL` | §5 |
| `backend/src/app/orchestrator/requesters/MessageChannelRequester.ts` | CRUD; credential `.strip()` + `?? null` | §5 |
| `backend/src/app/validation/joi_rules.ts` | `isCustomerOwnedMessageChannel` | §5 |
| `backend/requesters.website.ts` | `customer.messageChannel` map | §5 |
| `backend/src/app/orm/models/Organization.ts` | `hasMany MessageChannel` + mixins | §3.4 |
| `backend/src/resources/trans/ar/general.ts` | `messageChannelType` / `messageChannelStatus` AR | §3.5 |
| `backend/src/resources/trans/en/general.ts` | EN mirrors | §3.5 |
| `backend/src/app/gql/definitions/base.graphql` | type/status GQL enums | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | `_MessageChannel` + `_MessageChannelFilter` (`type`+`status`) | §4 |
| `backend/src/app/gql/bridges/customer/MessageChannelBridge.ts` | filter map + nest parents | §4–§6 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | `GetOneParent` includes `MessageChannelModel` | §4 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register + resolvers pass `filter` | §4 |
| `website/` form + directory | portal UI | `flow-customer-message-channels.md` |
| `.cursor/rules/message-channel-domain.mdc` | Channel invariants | governance |
| `.cursor/rules/requester-type-conditional-strip.mdc` | Unused credential `.strip()` | §5 |

## Related

- `docs/platforms/backend/contracts/message-template-domain.md` (template kinds + optional channel FK)
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md` (optional template FKs on Meeting)
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/website/flow-customer-message-channels.md`
- `.cursor/rules/organization-tenant-ownership.mdc`
- `.cursor/rules/message-channel-domain.mdc`
