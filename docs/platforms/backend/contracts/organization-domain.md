# Organization Domain Contract (Current)

## 1) Scope

Current Ejtmaa organization (tenant) surface:

- ORM persistence for one organization per customer account,
- customer GraphQL read of the current actor's organization,
- supervisor GraphQL list/detail reads and nested customer relation,
- localization for organization status,
- demo seed via console `test_seed` (not `init`),
- seed logo assets under `static/upload/__seed/images/`,
- website GraphQL mirrors for the customer organization read.

Out of scope for this contract (not shipped):

- organization requesters / write mutations,
- member domain (see `member-domain.md` — shipped separately),
- meeting domain,
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

`Customer` remains the **actor** model. This change set only adds the organization association; it does not alter Customer attrs, abilities, or user/notification ownership.

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
| `getOrganization` | used by customer GQL root-one `{ me: true }` |
| `setOrganization` | association setter |
| `createOrganization` | seed path (`SeedPatch.seedDemoOrganizations`) — omit key `"customer_id"` |

Runtime consumers of these mixins:

- Customer GraphQL `organization` → framework calls `context.customer.getOrganization(...)`
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

### Root query

- `organization: _Organization`

Resolver (`CustomerSchema`):

```ts
prepareOneGQLModel({ me: true })
```

Bridge: `backend/src/app/gql/bridges/customer/OrganizationBridge.ts`

- `ident = "organization"`, `typeIdent = "_Organization"`, `ormModel = OrganizationModel`
- `GetOneParent = { me: true }`
- no `getRootOrmParent` / `getOrmFindOptions` overrides

Resolution path:

1. `CustomerBridgeBase.getRootOrmParent({ me: true })` returns `context.customer` (throws `NOT_PERMIT` if absent).
2. `BridgeBase.getOneModel` calls `customer.getOrganization(...)` using singular accessor from `Static.ident`.

Registered in `CustomerSchema.registeredBridges` with `MeBridge` and `NotificationBridge`.

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

No website organization UI/adapters in this change set (customer mirrors only). No cpanel organization UI.

## 9) Failure / auth modes (read path)

| Surface | Condition | Behavior |
|---|---|---|
| Customer `organization` | no `context.customer` | `NOT_PERMIT` from role base `getRootOrmParent` |
| Customer `organization` | customer has no org row | framework `404` from `getOneModel` empty accessor |
| Supervisor `organization(id)` | missing id | framework `404` |
| Supervisor lists | unbounded | prevented by role `withListable` max length |

## 10) Traceability map

| Path | Role | Section |
|---|---|---|
| `backend/src/app/orm/models/Organization.ts` | ORM source of truth | §3–§3.4 |
| `backend/src/app/orm/models/Customer.ts` | `hasOne Organization` + org mixins (`get`/`set`/`create`) | §3.5 |
| `backend/src/resources/trans/ar/general.ts` | AR enum labels | §4 |
| `backend/src/resources/trans/en/general.ts` | EN enum labels | §4 |
| `backend/src/app/gql/definitions/base.graphql` | Shared status wrappers | §4 |
| `backend/src/app/gql/definitions/customer.graphql` | Customer `_Organization` + root query | §5 |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | Customer me-bound root-one | §5 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | Register + resolver | §5 |
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

- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/supervisor-admin-read-surfaces.md`
- `docs/platforms/backend/contracts/supervisor-customers-and-stats.md`
- `docs/platforms/backend/patterns/scheduler-console-seed-db.md`
- `docs/invariants/backend.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `docs/platforms/cpanel/graphql-mirror-and-tooling.md` (deferred until `cpanel/` restored)
