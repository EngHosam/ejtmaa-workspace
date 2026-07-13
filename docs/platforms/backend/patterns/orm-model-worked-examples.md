# ORM Model Worked Examples (Ejtmaa)

## Example 1: Actor Model (`CustomerModel`)

Reference: `backend/src/app/orm/models/Customer.ts`

### Why This Is Actor-Path

- Owns user identity relation through polymorphic owner linkage.
- Owns notification relations.
- Exposes business permission through `can()` for notification actions.

### Structural Highlights

- `Attrs` includes persisted fields and virtual URL field.
- `attributes()` defines:
  - identity fields,
  - validated email field,
  - virtual `avatar_url`.
- `initOptions()` defines unique email index.
- `boot()` defines scoped owner relations to `User` and `Notification`.

### Ability Highlights

- `Ability.NOTIFICATION.sub = "deleteAll"` branch.
- Validates owner type and owner id before allowing operation.
- Uses `this.iCan(...)` for consistent denial behavior.

### ORM ↔ GQL Touchpoints

- Queried through role bridge path:
  - `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`
  - `backend/src/app/gql/bridges/customer/MeBridge.ts`
- Root list/order requirements rely on `updated_at` and bounded list settings in bridge base.

### Reuse Template (Actor)

Use this shape when model:

- represents role owner or owner-scoped aggregate,
- requires domain ability checks,
- participates in role-scoped bridge retrieval.

## Example 2: Non-Actor Model (`TokenModel`)

Reference: `backend/src/app/orm/models/Token.ts`

### Why This Is Non-Actor Path

- Represents authentication token storage, not business actor entity.
- No dedicated `Ability` branches.
- Domain behavior is utility-focused (`getPublicToken`, `getPrivateToken`).

### Structural Highlights

- `Attrs` defines token primary key and optional payload.
- `attributes()` uses JSONB payload with typed-object metadata.
- `initOptions()` keeps minimal model/table identity contract.
- `boot()` defines direct `belongsTo(User)` relation.

### Helper Highlights

- `getPrivateToken(publicToken)` verifies JWT wrapper and extracts private token.
- `getPublicToken()` issues public token with expiry.
- Helper logic is deterministic and scoped to token behavior.

### ORM ↔ GQL Touchpoints

- Not directly exposed as a first-class GraphQL entity in current role schemas.
- Still must keep stable relation contracts since auth/requesters depend on user-token traversals.

### Reuse Template (Non-Actor)

Use this shape when model:

- stores infrastructure/security support data,
- has no standalone business ability matrix,
- is consumed via requesters/middleware rather than direct role bridge exposure.

## Choosing Between the Two

Pick actor path if:

- operation permissions depend on owner identity and domain state.

Pick non-actor path if:

- model is technical/supportive and permission decisions are delegated to caller context.

When uncertain:

1. Start from non-actor.
2. Add actor contracts only when explicit ability branches become necessary.
