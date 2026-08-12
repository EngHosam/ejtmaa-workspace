# Ejtmaa Backend Invariants

## Purpose

Practical invariants for the current Ejtmaa backend. Scoped to active modules: Customer, Organization, Member, MessageTemplate, Meeting, MeetingParticipant, AgendaItem, Decision, Vote, Supervisor, User, Token, Notification, SystemSetting.

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

Shared enums in `base.graphql` only. Mirror sync to frontends is command-based copy when the target platform checkout exists:
- `website/src/types/gql/**` (active)
- `cpanel/src/types/gql/**` (deferred while `cpanel/` checkout is temporarily absent — do not sync)

## B15. GraphQL Relation Cardinality Gate Invariant

Add relations only when expected count is `<= 100`. Clarify ambiguous cases before schema expansion.

- High-cardinality collections (example: organization members) use **root list/detail** with `withListable`, not nested `parent.many`.
- Cardinality-safe inverses (`belongsTo` / expected count 1) may be nested (examples: `_Member.organization`, `_MessageTemplate.organization`, `_MessageTemplate.messageChannel` (optional), `_Meeting.organization` / `chairperson` / template refs).
- When adding any nested SDL relation, update the preparing bridge's `GetOneParent` / `GetManyParent` (see `.cursor/rules/gql-root-parent-payload-contract.mdc` §5).
- Customer org-owned children that resolve `{ me: true }` to the customer's Organization must extend `CustomerOrganizationOwnedBridgeBase` — do not re-copy that `getRootOrmParent` per entity bridge.
- Meeting invite timing is `notify_start_at` (independent of lifecycle `status`); invite progress is `notify_status`. Do not derive notify start from status transitions alone.

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

## B22. Organization Tenant Ownership Invariant

`Organization` is a non-actor tenant owned by exactly one `Customer` via `customer_id`:

- real FK constraint (do not use `constraints: false` for this single-owner link),
- unique index on `customer_id` (one org per customer),
- ownership column name is `customer_id`, not a polymorphic `owner_*` pair.

## B23. Me-Bound HasOne Root Query Invariant

When a customer-role root-one resolves a `hasOne` from the authenticated customer (`Static.ident` matches association key), use `prepareOneGQLModel({ me: true })` and let role base + framework accessor resolve the row.

Do not invent `as` aliases or override `getRootOrmParent` / `getOrmFindOptions` unless the association key differs from bridge `ident` or scoping cannot use the default accessor.

## B24. Organization Host Identification Is Not Authentication

An organization id supplied by a client — `organizationid` HTTP header (`org_host` middleware) or optional meeting-socket handshake `organizationId` — identifies **which tenant a request claims**, nothing more. The id is public: `POST /website/custom/org/start` returns it to any visitor on an organization host.

Consequences:

- HTTP `org_host` resolution always requires `status === "ACTIVE"` and throws `404` when it does not resolve,
- optional meeting handshake `organizationId` is defense-in-depth against `meeting.organization_id`; it is **not** sufficient alone to join `/meeting`,
- `/meeting` proof is `MeetingAuthenticationIOMiddleware`: `memberId` + `memberToken` (`Member.access_token`) + `meetingId` + roster row + ACTIVE organization,
- actor namespaces (`/customer`, `/supervisor`) authenticate through the handshake `token` in `AuthenticationIOMiddleware`,
- any other organization-scoped surface that needs actor trust must add its own credential on top of a public organization id.

Contracts: `docs/platforms/backend/contracts/client-portal-http-website.md`, `docs/platforms/backend/contracts/meeting-realtime-socket.md` §2, `docs/platforms/website/organization-host-routing.md` §8.

## B25. Live Session State Is Not A Public Column

A collaborative document persisted as a BLOB (`meetings.live_state`) is **internal transport state**, not a readable field.

Consequences:

- the column must be excluded from the bridge's auto-registered attrs (`registerOrmAttrs = { expect: [...] }`); the same applies to every ORM column that must never leave through GraphQL,
- while a session is live, the document owns collaborative map fields and most SQL meeting columns stay stale by design; readers of those columns must not be told otherwise,
- reflecting durable fields from the live document onto SQL happens in **one explicit transactional helper** (`completeMeetingLiveToSql` in `MeetingLiveDocHelper`) on **`meeting.live.complete` only** — never as a side effect of `meeting.live.update`,
- **Lifecycle SQL exception (narrow):** dedicated inbound events `meeting.live.start` and `meeting.live.complete` may write SQL. `start` writes meeting `status → STARTED` immediately (registry row bound to the same instance). `complete` runs the full durable reflect then clears `live_state` and destroys the in-memory doc after commit. `meeting.live.update` must not write SQL columns,
- the document codec is fixed (Yjs V2 on the BLOB, on the wire, and on both ends); mixing V1 and V2 throws at runtime, so the `yjs` version stays pinned and equal in `backend/` and `website/`.

Contracts: `docs/platforms/backend/contracts/meeting-live-state.md` §6, `docs/platforms/backend/contracts/meeting-realtime-socket.md` §3.

## B26. Time-Window Policy Is Model-Declared And Gate-Enforced

A business time window (minimum lead before an event, a freeze before a side effect starts) is a domain rule, not field decoration.

Consequences:

- the window is a **static on the owning model** (`MeetingModel.MIN_LEAD_MS`, `MeetingModel.NOTIFY_MIN_GAP_MS`); Joi rules, `can(...)`, and requesters read that static instead of re-declaring the arithmetic,
- every write path that can violate the window checks it — including transitions that accept **no** input for the guarded field (approve re-checks the lead because it never receives a `datetime`),
- a timestamp column validated against another (`notify_start_at` vs `datetime`) is checked on every write path, and readers must tolerate `null` on rows written before the column existed,
- when a transition invalidates cached or collaborative state, the reset is part of the same transaction and the memory eviction runs in `transaction.afterCommit` — never before the commit, and never with a flush that would rewrite what the transaction cleared.

Contracts: `docs/platforms/backend/contracts/meeting-domain.md` §3.2b, §9.1a; `docs/platforms/backend/contracts/meeting-live-state.md` §3.1.
