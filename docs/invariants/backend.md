# Ejtmaa Backend Invariants

## Purpose

Practical invariants for the current Ejtmaa backend. Scoped to active modules: Customer, Supervisor, User, Token, Notification, SystemSetting.

Interpretation rule:
- invariants below define backend obligations for the Ejtmaa product surface.

## B0. Repository Ownership

`backend/` is its own git repository inside the workspace.
Git operations for backend files must run from `backend/`, not from nested platform repos or by assuming the workspace root owns those files.

## B1. Requester Decorator Invariant

All requester classes must use `@requester("ident")`, and all callable use-case methods must use `@sub(...)`.

## B2. Requester Transaction Invariant

Multi-step write use cases must use one requester-level transaction from `startTransaction()`.

## B3. Requester Validation Invariant

Every requester method must validate input and facts before execution.

## B4. Requester Result Invariant

Requester methods must return `SJsonResult` and not raw domain objects.

## B5. GraphQL Contract Sync Invariant

Any `.graphql` definition change must be followed by `yarn generate-types` and `yarn type-check`.

## B6. Bridge Registration Invariant

Any bridge required to resolve an exposed GraphQL relation must be present in `registeredBridges` for that role schema.

## B7. Role Context Invariant

Role schema execution must use matching role context:
- customer schema with customer context
- supervisor schema with supervisor context

## B8. Authorization Three-Level Invariant

Authorization is layered:
1. route/sub actor restriction
2. facts validation
3. model ability check (for entity-sensitive actions)

## B9. Model Factory Invariant

All model usage goes through exported model factory functions.

## B10. JSONB Field Typing Invariant

Use JSONB flags consistently:
- generic JSON object => `isTypedObject: true`
- multilingual single object => plain JSONB (no typed-object flag)

## B11. Scheduler Safety Invariant

Scheduled tasks must be idempotent, observable, and explicitly registered in scheduler config.

## B12. English Documentation Invariant

Persisted docs under `/docs` must remain in English.

## B13. Enum Contract Invariant

Enum-like business fields follow the project enum contract across model, requester, GraphQL, and localization.

## B14. Shared GraphQL Enum and Mirror Sync Invariant

Shared enums in `base.graphql` only. Mirror sync to frontends is command-based copy:
- `website/src/types/gql/**`
- `cpanel/src/types/gql/**`

## B15. GraphQL Relation Cardinality Gate Invariant

Add relations only when expected count is `<= 100`. Clarify ambiguous cases before schema expansion.

## B16. GraphQL Type Block Layout Invariant

Role model object types follow consistent block layout (`# info`, `# timestamps`, `# relations`, `# pagination`).

## B17. Role-Local Relation Ownership Invariant

Relations declared on the role contract that owns/consumes them. No symmetry copying across roles.

## B18. GraphQL Relation Naming and Exposure Invariant

Relation field names mirror ORM relation names. Do not expose `*_id` columns when relation object exists.

## B19. Multilingual Text Contract Invariant

Multilingual domain text uses `MultiLangString` at ORM level; GraphQL exposes localized `String`.

## B20. Principal Guard Placement Invariant

Guard ownership stays single-source per parent mode. Do not duplicate `willPrepare` when role base already enforces the guard.

## B21. GraphQL Enum Output Wrapper Invariant

Enum-like output fields use object wrappers (`type _X { value: _XValue!, label: String }`) in `base.graphql`.
