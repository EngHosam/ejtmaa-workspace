# Organization Domain Contract (Current)

## 1) Scope

Current Ejtmaa organization (tenant) surface:

- ORM persistence for one organization per customer account,
- customer GraphQL read of the current actor's organization,
- supervisor GraphQL list/detail reads and nested customer relation,
- localization for organization status,
- demo seed via console `test_seed` (not `init`),
- seed logo assets under `static/upload/__seed/images/`,
- website GraphQL mirrors for the customer organization read,
- website customer write surface via `OrganizationRequester` (`read` | `upsert`) — see §9,
- website organization settings form — `docs/platforms/website/flow-customer-organization.md`.

Out of scope for this contract (not shipped):

- accepting or writing `custom_domain` from the customer organization form/requester,
- customer control of `status` on upsert (create always forces `ACTIVE`; update never writes status),
- member domain (see `member-domain.md` — shipped separately),
- message template domain (see `message-template-domain.md` — shipped separately),
- meeting domain (see `meeting-domain.md` — shipped separately),
- supervisor organization stats aggregate,
- cpanel GraphQL mirrors / UI (cpanel platform checkout is temporarily absent; do not sync supervisor mirrors until restored).

## 2) Domain purpose

`Organization` is the tenant entity owned by a `Customer` account.

- `Customer` remains the login actor and account controller.
- `Organization` holds tenant identity (name, branding, domains, status).
- Ownership is single-parent via `customer_id` (not polymorphic `owner_type` / `owner_id`).

Cardinality: **one organization per customer** (unique index on `customer_id`).

## 3) ORM model

File: `backend/src/app/orm/models/Organization.ts`

Classification: **non-actor** (`Model<Attrs, Omit<Attrs, "id">>` — no `Ability`, no `can()`).

Persistence names:

- `modelName`: `organization`
- `tableName`: `organizations`

### 3.1 Attrs layout

Attrs are grouped as:

- `//relations` — `customer_id`
- `//info` — persisted identity fields
- `//virtual` — `logo_url`

`attributes()` field order matches that Attrs grouping.

### 3.2 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `customer_id` | BIGINT | no | FK to `customers.id`, real constraint |
| `name` | STRING(191) | no | |
| `description` | TEXT | yes | |
| `logo_file` | STRING(191) | yes | relative upload path |
| `primary_color` | STRING(191) | yes | brand hex |
| `secondary_color` | STRING(191) | yes | brand hex |
| `subdomain` | STRING(191) | yes | case-insensitive unique when set |
| `custom_domain` | STRING(191) | yes | case-insensitive unique when set |
| `status` | STRING(191) | no | enum `organizationStatus`, default `ACTIVE` |
| `logo_url` | VIRTUAL | — | derived via `getImagePath(logo_file)` |

### 3.3 Indexes

- `organizations_unique_customer` — UNIQUE(`customer_id`)
- `organizations_unique_subdomain` — UNIQUE(lower(subdomain))
- `organizations_unique_custom_domain` — UNIQUE(lower(custom_domain))

PostgreSQL unique indexes allow multiple NULL subdomain/custom_domain values.

### 3.4 Relations — `Organization` side

File: `backend/src/app/orm/models/Organization.ts` → `boot()`

- `belongsTo(Customer)` on `customer_id` / `targetKey: id`
- **real FK constraint** (`constraints` not disabled — unlike polymorphic `User` / `Notification` owner links)

Mixins on `OrganizationModel`:

- `customer?`
- `getCustomer` / `setCustomer` / `createCustomer`

### 3.5 Relations — `Customer` side (modified)

File: `backend/src/app/orm/models/Customer.ts`

`Customer` remains the **actor** model. Organization association mixins live here. `Customer.Ability.ORGANIZATION` (`read` | `upsert`) and `can("ORGANIZATION")` are the authorization surface for the website organization requester — see §9.

`Customer.boot()` addition:

```ts
this.hasOne(Organization(), {
    sourceKey: "id",
    foreignKey: "customer_id"
});
```

Notes:

- real FK (no `constraints: false`)
- association key defaults to `organization` (matches `Organization.initOptions().modelName` and GQL bridge `ident`)
- inverse of `Organization.belongsTo(Customer)`

Declared mixins (organization section):

| Mixin | Role |
|---|---|
| `organization?` | loaded association |
| `getOrganization` | nested `_Me.organization`, org-owned GQL roots, form requester |
| `setOrganization` | association setter |
| `createOrganization` | seed path (`SeedPatch.seedDemoOrganizations`) — omit key `"customer_id"` |

Runtime consumers of these mixins:

- Customer GraphQL `_Me.organization` → Sequelize `hasOne` include via registered `OrganizationBridge` (parent `CustomerModel`)
- Org-owned customer roots (`members`, `meetings`, …) → `CustomerOrganizationOwnedBridgeBase` calls `getOrganization()`
- Supervisor nested `_Customer.organization` → Sequelize `hasOne` include via registered `OrganizationBridge`
- Demo seed → `customer.createOrganization({ ... })` for a subset of seeded customers

## 4) Status enum and localization

Enum identify: `organizationStatus`

Values:

| Value | AR label | EN label |
|---|---|---|
| `ACTIVE` | نشط | Active |
| `DISABLED` | معطّل | Disabled |

Sources:

- `backend/src/resources/trans/ar/general.ts` → `enums.organizationStatus`
- `backend/src/resources/trans/en/general.ts` → `enums.organizationStatus`

ORM wiring:

- `OrganizationStatus = keyof G_Tr["enums"]["organizationStatus"]`
- column `enum: ["one", "organizationStatus"]`, `isTypedObject: true`

GraphQL shared wrappers in `backend/src/app/gql/definitions/base.graphql`:

- `enum _OrganizationStatusValue { ACTIVE DISABLED }`
- `type _OrganizationStatus { value: _OrganizationStatusValue!, label: String }`

## 5) Customer GraphQL surface

SDL: `backend/src/app/gql/definitions/customer.graphql`

### Type `_Organization`

Implements `_Timestamps`.

Exposed fields:

- info: `id`, `name`, `description`, `logo_file`, `logo_url`, `primary_color`, `secondary_color`, `subdomain`, `custom_domain`, `status`
- timestamps: `created_at`, `updated_at`

Not exposed on customer role:

- `customer_id` column (ownership is implicit via me-bound root)
- nested `customer` relation

### Nested on `_Me` (no customer root query)

- `_Me.organization: _Organization` — auto-wired from `Customer.hasOne(Organization)` (association key `organization` → `OrganizationBridge.ident`).
- There is **no** customer root `Query.organization`. Actor-scoped org reads for the portal go through `me { organization { … } }` (or the form requester).

Bridge: `backend/src/app/gql/bridges/customer/OrganizationBridge.ts`

- `ident = "organization"`, `typeIdent = "_Organization"`, `ormModel = OrganizationModel`
- `GetOneParent = CustomerModel | MemberModel | MessageTemplateModel | MeetingModel` (nested parents only — no `{ me: true }` root-one)
- no `getRootOrmParent` / `getOrmFindOptions` overrides

Resolution path for `_Me.organization`:

1. `MeBridge` loads the current `Customer` row.
2. Framework includes / resolves association `organization` via `OrganizationBridge` with parent `CustomerModel`.
3. Missing org row → GraphQL `null` (not a root 404).

Registered in `CustomerSchema.registeredBridges` (still required for Me + Member/Meeting/MessageTemplate nests).

## 6) Supervisor GraphQL surface

SDL: `backend/src/app/gql/definitions/supervisor.graphql`

### Nested relation on customer

`_Customer.organization: _Organization` — resolved via `Customer.hasOne(Organization)` when `OrganizationBridge` is registered.

### Type `_Organization`

Implements `_Timestamps & _Pagination`.

Same info + timestamp fields as customer `_Organization`, plus:

- relations: `customer: _Customer` (not `customer_id`)
- pagination: `total_count: Int`

### Filter / sort

`_OrganizationFilter`:

- `search` — iLike across `name`, `description`, `subdomain`, `custom_domain`
- `status` — `_OrganizationStatusValue`
- `customerId` — exact `customer_id`
- `sort` — `_OrganizationSortValue`

`_OrganizationSortValue`:

- `UPDATED_AT_DESC` (default)
- `UPDATED_AT_ASC`
- `NAME_ASC`
- `NAME_DESC`

Secondary order always includes `id`.

### Root queries

- `organizations(filter: _OrganizationFilter): [_Organization]`
- `organization(id: ID!): _Organization`

Bridge: `backend/src/app/gql/bridges/supervisor/OrganizationBridge.ts`

- root many: listable + replacements + filter where + deterministic order
- root one: `where: { id }`

`SupervisorSchema` registers `OrganizationBridge` and wires both resolvers with explicit `{ filter }` / `{ id }` parents.

## 7) Seed and console (mechanism only)

Demo payload values (emails, passwords, org names, colors, status samples) live in code — not documented here.

### `SeedPatch.init()`

Seeds system settings and default supervisor when none exists.

Does **not** seed customers or organizations.

### `SeedPatch.update("test_seed")`

Entry: `DatabaseConsole.update` choice `test_seed` → `SeedPatch.update("test_seed")`.

Order:

1. `seedDemoCustomers`
2. `seedDemoOrganizations`

Each method early-returns when the target table already has rows (`count() > 0`); no upsert.

Organizations are created via `customer.createOrganization(...)` for a subset of seeded customers.

### Seed logo assets

Directory: `backend/static/upload/__seed/images/`

Whitelisted by `backend/static/upload/.gitignore` (`!__seed/`, `!__seed/**`).

Referenced from seed rows as `logo_file` paths under `__seed/images/`.

## 8) Frontend mirrors and verification

| Platform | Mirrored artifacts | Current status |
|---|---|---|
| `website/` | `definitions/base.graphql`, `definitions/customer.graphql`, `gql-types/base.ts`, `gql-types/customer.ts` | Active — sync after backend `yarn generate-types` |
| `cpanel/` | Would be `base` + `shared` + `supervisor` SDL/types under `src/types/gql/**` | **Deferred** — `cpanel/` checkout temporarily removed; **do not** create or sync supervisor GQL mirrors until the platform folder returns |

Do not hand-edit website mirrors.

Backend verification scripts (existing):

- `yarn generate-types`
- `yarn type-check`

Website organization **settings form** (requester-backed, not a GQL adapter list): `docs/platforms/website/flow-customer-organization.md`. No cpanel organization UI.

## 9) Write surface (`OrganizationRequester`)

HTTP: existing customer form controller → `API.FORMS.CUSTOMER.R("organization")(sub)` → `/forms/customer/requester/organization/:sub`. No new controller.

### 9.1 Ability — `Customer.Ability.ORGANIZATION`

File: `backend/src/app/orm/models/Customer.ts`

```ts
ORGANIZATION: {
  sub: "read" | "upsert"
}
```

| Sub | Guard |
|---|---|
| `read` | Customer may always read their tenant profile (empty defaults when no org row) |
| `upsert` | Customer may always upsert their tenant profile (create when missing; update when present) |

`can("ORGANIZATION")` allows both subs without requiring an existing organization row (create path must work).

### 9.2 Requester subs

File: `backend/src/app/orchestrator/requesters/OrganizationRequester.ts` (`@requester("organization")`, platform `website`, actor `customer`).

| Sub | Behavior |
|---|---|
| `read` | Facts → `can ORGANIZATION read` → if org: values `{ name, description, logo_file, primary_color, secondary_color, subdomain }`; if none: empty defaults (`name: ""`, nullables `null`). **No `status` / `custom_domain` in values.** |
| `upsert` | Validate fields → start transaction → `can ORGANIZATION upsert` → if org: `organization.update(payload)`; else `customer.createOrganization({ ...payload, status: "ACTIVE" })`. Never writes `custom_domain` or `status` on update. |

Field rules (requester-local):

| Field | Rule |
|---|---|
| `name` | `joi.string().trim().min(2)` — keep internal spaces |
| `description` | optional nullable string; `""` / null → null |
| `logo_file` | optional string filename; `""` / null → null on persist |
| `primary_color` / `secondary_color` | optional nullable string; `""` / null → null (no hex schema invented) |
| `subdomain` | **required**; `trim` + `lowercase` + `min(4)`; `prepareCleanString` with `cleanWhiteSpaces`; English alpha only (`Validator.isAlpha(..., "en-US")`) → `subdomain.wrong`; reserved / bad-word substring list → `subdomain.invalid`; case-insensitive uniqueness excluding current org id → `subdomain.registered` |

Reserved subdomain list: `backend/src/app/helpers/BadWords.ts` spread into `notAllowedSubdomains` in the requester (platform mounts, product terms, infra, brands). Match uses `.includes(s)` (substring), same family as historical company-register validation.

Joi keys live under `backend/src/resources/trans/{ar,en}/general.ts`: `subdomain.registered`, `subdomain.wrong`, `subdomain.invalid`.

### 9.3 Maps + messages

| Map | Entry |
|---|---|
| `backend/requesters.website.ts` | `customer.organization: "read" \| "upsert"` |
| `website/src/types/requesters/requesters.website.ts` | Same union (W18) |

Messages (`backend/src/resources/trans/{ar,en}/messages.ts`):

- create branch → `SUCCESS_CREATE`
- update branch → `SUCCESS_UPDATE`

### 9.4 Failure modes (write)

| Condition | Result |
|---|---|
| Subdomain too short / missing | Joi `min` / required |
| Non-English-alpha subdomain | `subdomain.wrong` |
| Forbidden / reserved substring | `subdomain.invalid` |
| Subdomain taken by another org | `subdomain.registered` |
| Other validation errors | field / firewall errors via `throwIfNotValid` |

Verify: `yarn type-check` in `backend/`.

## 10) Failure / auth modes (GQL read path)

| Surface | Condition | Behavior |
|---|---|---|
| Customer `_Me.organization` | no `context.customer` | `MeBridge.willPrepare` → `NOT_PERMIT` |
| Customer `_Me.organization` | customer has no org row | GraphQL `null` |
| Org-owned roots (`members`, …) | customer has no org row | `404` from `CustomerOrganizationOwnedBridgeBase` |
| Supervisor `organization(id)` | missing id | framework `404` |
| Supervisor lists | unbounded | prevented by role `withListable` max length |

Note: the **form** `read` sub does **not** 404 when no org exists — it returns empty defaults so the settings page can create on first upsert.

## 11) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Organization.ts` | ORM source of truth | §3–§3.4 |
| `backend/src/app/orm/models/Customer.ts` | `hasOne Organization` + org mixins + `Ability.ORGANIZATION` | §3.5, §9.1 |
| `backend/src/app/orchestrator/requesters/OrganizationRequester.ts` | `read` / `upsert` | §9 |
| `backend/src/app/helpers/BadWords.ts` | Reserved/bad-word list for subdomain | §9.2 |
| `backend/requesters.website.ts` | `customer.organization` union | §9.3 |
| `website/src/types/requesters/requesters.website.ts` | W18 mirror | §9.3 |
| `docs/platforms/website/flow-customer-organization.md` | Website settings form | §8, §9 |
| `backend/src/resources/trans/ar/general.ts` | AR enum labels | §4 |
| `backend/src/resources/trans/en/general.ts` | EN enum labels | §4 |
| `backend/src/app/gql/definitions/base.graphql` | Shared status wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | Customer `_Organization` + `_Me.organization` (no root) | §5 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | Nested Me/Member/Meeting/MessageTemplate | §5 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register bridge (no root org resolver) | §5 |
| `backend/src/app/gql/definitions/supervisor.graphql` | Supervisor type/filter/queries + customer relation | §6 |
| `backend/src/app/gql/bridges/supervisor/OrganizationBridge.ts` | Supervisor many/one filters | §6 |
| `backend/src/app/gql/schemas/SupervisorSchema.ts` | Register + resolvers | §6 |
| `backend/src/app/gql/gql-types/base.ts` | Generated | §8 |
| `backend/src/app/gql/gql-types/customer.ts` | Generated | §8 |
| `backend/src/app/gql/gql-types/supervisor.ts` | Generated | §8 |
| `backend/src/app/orm/patches/SeedPatch.ts` | init vs `test_seed` mechanism | §7 |
| `backend/src/console/DatabaseConsole.ts` | `test_seed` choice | §7 |
| `backend/static/upload/__seed/images/org-*.svg` (5 SVGs) | Seed logo assets; row values stay in `SeedPatch` | §7 |
| `website/src/types/gql/definitions/base.graphql` | Customer mirror | §8 |
| `website/src/types/gql/definitions/customer.graphql` | Customer mirror | §8 |
| `website/src/types/gql/gql-types/base.ts` | Customer mirror | §8 |
| `website/src/types/gql/gql-types/customer.ts` | Customer mirror | §8 |
| `website/lib/tsconfig.tsbuildinfo` | Build cache noise | excluded from narrative |
| `cpanel/src/types/gql/**` | Supervisor mirrors | deferred — platform folder temporarily absent; not synced |
| `backend/.types/models.ts` | Generated model registry (gitignored; regenerated at boot) | excluded from narrative |

## Related

- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/message-template-domain.md`
- `docs/platforms/backend/contracts/meeting-domain.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/patterns/scheduler-console-seed-db.md`
- `docs/invariants/backend.md`
- `docs/platforms/website/flow-customer-organization.md`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `docs/platforms/cpanel/graphql-mirror-and-tooling.md` (deferred until `cpanel/` restored)
