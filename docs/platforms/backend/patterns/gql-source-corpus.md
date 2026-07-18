# GQL Source Corpus (Ejtmaa)

## Scope

Canonical source: `backend/src/app/gql/**`

Framework contracts: `@nodejs/gql` (`BridgeBase`, `GQLSchemaBase`, `GQLProvider`)

## Domain classification

### Canonical (must follow)

- Role-scoped schemas and role bridge bases
- Deterministic list strategy
- Depth/include guardrails
- Bridge lifecycle through framework factories

### Drift (block)

- Hardcoded principal loading in `getEnvContext`
- Mixed resolver typing (`any` vs generated types)
- Inconsistent parent-mode flags without mapping
- Unsafe SQL interpolation in `literal`

## Function evidence

### Schemas

- `backend/src/app/gql/schemas/CustomerSchema.ts`
- `backend/src/app/gql/schemas/SupervisorSchema.ts`

### Role bridge bases

- `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts`
- `backend/src/app/gql/bridges/customer/CustomerOrganizationOwnedBridgeBase.ts` — org-owned `me` → Organization
- `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts`

### Entity bridges

| Bridge | Role |
|---|---|
| `backend/src/app/gql/bridges/customer/MeBridge.ts` | Customer profile |
| `backend/src/app/gql/bridges/customer/NotificationBridge.ts` | Customer notifications |
| `backend/src/app/gql/bridges/customer/OrganizationBridge.ts` | Customer organization + inverse parents |
| `backend/src/app/gql/bridges/customer/MemberBridge.ts` | Org members (extends org-owned base) |
| `backend/src/app/gql/bridges/customer/MessageTemplateBridge.ts` | Org message templates (extends org-owned base) |
| `backend/src/app/gql/bridges/customer/MeetingBridge.ts` | Org meetings (extends org-owned base) |
| `backend/src/app/gql/bridges/supervisor/MeBridge.ts` | Supervisor profile |
| `backend/src/app/gql/bridges/supervisor/NotificationBridge.ts` | Supervisor notifications |
| `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts` | Customer list/detail |
| `backend/src/app/gql/bridges/supervisor/CustomerStatsBridge.ts` | `customerStats.total_count` |
| `backend/src/app/gql/bridges/supervisor/OrganizationBridge.ts` | Supervisor organization list/detail |

### SDL

- `backend/src/app/gql/definitions/base.graphql`
- `backend/src/app/gql/definitions/customer.graphql`
- `backend/src/app/gql/definitions/supervisor.graphql`

On-disk draft SDL (reference copy):
- `backend/src/app/gql/definitions/shared.graphql`

## Expansion decision

- Write constitutions against canonical Ejtmaa style.
- Reject drift via blacklist + refusal matrix before implementation.

## Related

- `docs/platforms/backend/patterns/gql-pattern-atlas.md`
- `docs/platforms/backend/patterns/gql-constitution-map.md`
