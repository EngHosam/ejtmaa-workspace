# Models, Owners, Abilities, and Security

## 1) Model Layer Scope

Current active model files:
- `Model` (base extension)
- `User`
- `Token`
- `Customer`
- `Supervisor`
- `Notification`
- `SystemSetting`

All models use project factory exports (`ModelName = () => ormModel(...)`), consistent with the Ejtmaa model factory pattern.

## 2) Owner Model Topology

Owner types:
- `CUSTOMER`
- `SUPERVISOR`

Ownership mapping:
- `User` is polymorphic owner index (`owner_type`, `owner_id`).
- Owner models (`Customer`, `Supervisor`) each have:
  - scoped `hasOne(User)` relation
  - scoped `hasMany(Notification)` relation

This keeps auth identity and owner profile in separate tables while preserving polymorphic owner abstraction.

## 3) User and Token Contract

`User`:
- Stores login identity fields and owner linkage.
- Hashes `password` and `temporary_password` via setters.
- Supports owner scopes:
  - `owner_type_customer`
  - `owner_type_supervisor`
- `getOwner()` resolves owner by `owner_type`.

`Token`:
- Stores private token with `payload`.
- Public token is JWT wrapper around private token.
- `getPrivateToken(publicToken)` validates and extracts private token.

Auth middleware validates:
- token exists
- owner model exists
- email verified
- account not blocked

## 4) Notification Contract

`Notification` model:
- Polymorphic owner linkage (`owner_type`, `owner_id`).
- Enum-like fields:
  - `type` via `notificationType` translations
  - `identify` via `notificationIdentify` translations
- `data` as JSONB payload.

Owner helper pattern:
- `owner.notify(...)`
- `owner.markAllNotificationsAsSeen()`

Event emitters:
- `OnUserEvent`
- `OnCustomerEvent`

## 5) System Settings Contract

`SystemSetting` is key-value JSONB (`key`, `value.data`) and currently stores:
- Website content blocks:
  - `about`, `terms`, `privacy`, `rights`, `tips`
  - `faqs`, `videos`, `socials`
- Invoice/platform keys:
  - `is_vat_enabled`, `vat_percent`, `seller_name`, `vat_register_number`

Access methods:
- `setSetting`
- `getSetting`
- `getSettings`
- `removeSetting`

## 6) Abilities: Current State vs Required Pattern

### Current State

- Base `Model.can(...)` currently returns `"CAN"` by default.
- `User` overrides `can("LOGIN", ...)`.
- `Customer` and `Supervisor` define explicit `Ability` for notification deletion and override `can()`.
- `NotificationRequester.deleteAll` calls owner-level `can()` before mutation.

### Required Pattern (for future growth)

For owner-sensitive mutations, add explicit model-level abilities:
1. Define `Ability` type in model.
2. Override `can(to, props)` and use `this.iCan(...)`.
3. Use requester checks:
   - validate facts
   - call `owner.can(...)`
   - apply mutation only on `"CAN"`

This matches the Ejtmaa three-level authorization model:
- Route/decorator layer
- Facts layer
- Entity ability layer

## 7) Validation Security Layer

`joi_rules.ts` provides identity-safe model extraction:
- `Model().Opt(joi)` resolves numeric ID, select option, or model instance.
- Facts validators can auto-load linked `user` in external validation step.
- `isMobile`, `isMultiLang`, and actor fact validators centralize common constraints.

Guideline:
- Never trust route actor guard only.
- Always validate factual context in requester method.

## 8) HTTP Security Layer

`AuthenticationHttpMiddleware`:
- Initializes auth state per request.
- Reads token from headers.
- Enforces owner-type compatibility with route permission context.
- Exposes helpers:
  - `currentUser`, `currentOwner`, `currentCustomer`, `currentSupervisor`, `currentBy`

Middleware groups in express config enforce platform-specific granting:
- CPanel grants: `SUPERVISOR`

## 9) Data Type and JSONB Rules

Ejtmaa rules still apply:
- `MultiLangString` JSONB field: plain JSONB (no `isTypedObject`).
- Generic structured JSONB: use `isTypedObject: true`.
- Virtual URL fields should stay optional in model attrs (`avatar_url?`).

## 10) Recommended Next Hardening Steps

1. Expand owner abilities beyond notifications when new owner mutations are introduced.
2. Keep translation keys synchronized with thrown error keys as requesters evolve.
3. Keep enum translations synchronized whenever new enum-like fields are introduced.
4. Add explicit tests for ability-protected mutation flows.

## 11) Related Authoring References

Use these documents as the operational source for future model work:

- `docs/platforms/backend/patterns/orm-constitution-map.md`
- `docs/platforms/backend/patterns/orm-model-baseline-audit.md`
- `docs/platforms/backend/patterns/orm-model-authoring-standard.md`
- `docs/platforms/backend/patterns/orm-model-compliance-checklists.md`
- `docs/platforms/backend/patterns/orm-model-worked-examples.md`
