# Workspace Constitution (Full Reference)

This document preserves the full workspace constitution reference.
The always-on operational digest lives in `.cursor/rules/constitution.mdc`.

# CONSTITUTION
## Enterprise Mega Platform Architecture & Governance Constitution — Unified Ultra Edition
Version: 3.1
Last Updated: 2026-02-21
Status: Active
Authority: The Architect
Mode: Financial-Safe / Multi-Tenant / Drift-Resistant / Verification-Enforced / Mega-Scale

This Constitution defines immutable laws governing architectural boundaries, invariants, transactions, contracts, data evolution, security, observability, verification, and knowledge management across all platforms in this workspace.

It defines constraints and enforcement obligations — not implementations.

---

# I. AUTHORITY & LANGUAGE

## 1. Architectural Authority Law
The Architect is the final authority on:
- Architectural boundaries and layering
- Invariant definitions and enforcement models
- Consistency models and transactional boundaries
- Structural and dominant patterns
- Technology direction and stack choices
- Governance structure and review processes

Any structural deviation requires ADR and explicit approval.

## 2. Authoritative Language Law
All persisted authoritative knowledge must be written in English only.

English Required For:
- Constitution
- `/docs/**`
- Blueprints, Realizations, ADRs
- Invariants and Contracts
- Operational and Security standards

Arabic Permitted For:
- Discussion and business explanation
- User-facing content (UI/messages)
- Non-architectural business requirements (separate business docs)

---

# II. SOURCE OF TRUTH & NO-INVENTION

## 3. Repository Authority Law
The repository is the authoritative reference for:
- Structure and layering
- Naming and conventions
- Dominant patterns
- Build/verification/tooling scripts
- Contract definitions and schemas

New work must conform to the repository.
Deviation requires ADR.

## 4. Convention Extraction Law
Before introducing any new structure or approach, the executor must:
- Identify prevailing folder structure and boundaries
- Identify dominant patterns and naming
- Extend existing structure rather than redefine it

Repository consistency overrides preference.

## 5. No-Invention Law (Hard)
If architectural intent cannot be clearly derived from:
- Source code
- Established conventions
- Documented patterns

Then:
- No structural invention is permitted
- No paradigm shift is allowed
- Escalation is mandatory

Inference must not replace confirmation.

---

# III. LAYER MODEL & DEPENDENCY DIRECTION

## 6. Strict Layer Hierarchy Law
The workspace uses a strict knowledge and architecture layer model:

1. Constitution (this document)
2. Global Architecture (`docs/platforms/`)
3. Platform Blueprints (`docs/platforms/{platform}/overview.md`)
4. Platform Realizations (`docs/platforms`)
5. Domains (inside each Platform)
6. Use Cases & Flows
7. Data Models
8. Public Contracts (APIs/Schemas)
9. Events & Messaging
10. Integrations
11. Invariants
12. Resilience & Failure Models
13. Technology Stack
14. Governance
15. Knowledge Lifecycle
16. Process & Rituals
17. ADRs (`/docs/decisions`)
18. Session-only (never persisted)

Dependency Rule:
- Code and knowledge may only depend on layers below them.
- Cross-domain communication must occur through explicit public contracts.

## 7. Boundary Law (Hard)
Architectural boundaries are invariants.

No component may:
- Access internal state across boundaries directly
- Bypass invariant enforcement
- Leak infrastructure details across boundaries
- Mutate external state without explicit contract

Boundaries must survive technological change.

## 8. Dependency Direction Law (Hard)
Dependencies must be intentional.
- Reverse dependency is forbidden
- Cross-boundary interaction requires explicit contracts
- Architectural direction must not change for convenience

Ambiguity requires clarification.

---

# IV. INVARIANTS, CONSISTENCY, DETERMINISM

## 9. Invariant Law (Hard)
Invariants define system truth. They must be:
- Explicit
- Enforced
- Validated (tests or equivalent)
- Preserved during refactoring

Invariants must never be bypassed.

## 10. Invariant Levels & Locations
- Platform Invariants: `docs/invariants/{platform}.md` (currently `backend.md`, `website.md`, `cpanel.md`)
- Cross-platform operational digest: `.cursor/rules/constitution.mdc`

## 11. Consistency Boundary Law
Every operation must define a logical consistency boundary.
- Single consistency owner
- No competing ownership
- No undefined state transitions

Consistency must be explicit.

## 12. Deterministic Behavior Law (Hard)
Given the same state and input, behavior must be deterministic.

Hidden side effects are forbidden.
Non-determinism must be explicitly documented and justified.

Prohibited without justification:
- Uncontrolled randomness
- Time-based logic outside defined boundary
- Global mutable shared state
- Implicit side effects

---

# V. TRANSACTIONS, IDEMPOTENCY, CONCURRENCY

## 13. Transaction Ownership Law (Hard)
Transactions start and propagate only from defined orchestration roots.
- Each use case has a single transaction owner
- Transactions propagate downward through layers
- Side effects should use `afterCommit()` where possible
- Manual commit/rollback is prohibited unless an external side effect must be awaited; in that case commit before side effect, never rollback manually

Transaction boundaries align with use case boundaries.

## 14. Idempotency Law (Hard)
Externally triggered or retriable operations must be idempotent.
Repeated execution must not corrupt state or create duplicate side effects.

Applies to:
- Webhooks
- Payment callbacks
- Fulfillment
- Background jobs
- Retryable operations

## 15. External Interaction Law
External interactions must preserve invariants and consistency.
- Failure modes must be modeled
- Partial states must be controlled
- Implicit consistency assumptions are forbidden

## 16. Concurrency Control Law (Hard)
Concurrent operations must not violate invariants.
- Race conditions must be prevented by design
- Shared state mutation must be controlled
- Concurrency assumptions must never be implicit

---

# VI. FINANCIAL, FULFILLMENT, BACKGROUND

## 17. Financial Integrity Law (Hard)
Financial state transitions must be strongly consistent.
- Monetary mutations must be atomic
- Double processing must be prevented
- Balance-affecting operations must be auditable
- Rounding rules must be deterministic
- Financial invariants must not depend on external availability

Financial correctness overrides performance convenience.

## 18. Fulfillment Atomicity Law (Hard)
Fulfillment must be protected against duplication and loss.
- Idempotent
- Traceable delivery state
- Partial fulfillment explicitly modeled
- Retries must not duplicate delivery

## 19. Background Execution Law (Hard)
Background jobs must be:
- Idempotent
- Observable
- Retry-safe
- Concurrency-controlled

Jobs must not silently corrupt state.

---

# VII. AUDIT, SECURITY, OBSERVABILITY, ABUSE

## 20. Auditability Law
Material state transitions must be traceable.
Critical actions must leave reconstructable evidence.
Destructive actions must be attributable.

## 21. Observability Mandate (Hard)
Systems must provide:
- Structured logging
- Error visibility
- Operational insight

Silent failure is prohibited.
Changes must not reduce observability.

## 22. Rate & Abuse Protection Law
Externally exposed systems must protect against:
- Abuse
- Replay attacks
- Retry storms
- Resource exhaustion

Protective mechanisms must exist when exposure risk is present.

## 23. Sensitive Data Protection Law (Hard)
Sensitive data must:
- Never be logged unintentionally
- Be protected by design
- Be masked where unnecessary
- Be validated when externally received

## 24. Sensitive Codes & Fulfillment Compliance Law (Hard)
Card secrets and fulfillment data must be protected:
- Never log PIN/serial/credentials
- Mask in responses unless explicitly required and authorized
- Enforce uniqueness constraints where required
- Validate webhook payloads/signatures before acting
- Store only required fulfillment data (avoid duplication)

---

# VIII. MULTI-TENANCY, DATA EVOLUTION, CONTRACTS

## 25. Isolation Law (Hard)
Tenant boundaries must be strictly enforced.
Cross-tenant access is forbidden.
Isolation must be enforced by design, not convention.

## 26. Data Evolution Law
Data changes must:
- Preserve integrity
- Define migration strategy
- Avoid uncontrolled drift

## 27. Backward Compatibility Law (Hard)
Public contracts must not break without:
- Versioning strategy
- Migration plan
- Impact assessment
- ADR

Breaking change without governance is prohibited.
Contract drift is forbidden.

---

# IX. BLUEPRINT & PLATFORM GOVERNANCE

## 28. Platform Reuse Model
Reusable across all projects:
- Constitution
- Platform Blueprints

Project-specific:
- Platform Realizations
- Domains inside each platform

Blueprints define reusable patterns.
Realizations implement project-specific instances.

## 29. Blueprint Conformance Law (Hard)
Every Platform Realization must:
- Declare governing Blueprint
- Prove compliance with Blueprint invariants
- Record deviations via ADR (`/docs/decisions/`)

---

# XIII. VERIFICATION & RELEASE SAFETY

## 37. Verification Integrity Law (Hard)
A task is incomplete if any required verification check fails.

Verification executes checks only and must be derived strictly from what exists in `package.json`.

Rules:
- Only scripts explicitly present in `package.json` may be executed.
- If a check script does not exist, it must be skipped.
- If an existing check fails, the task is invalid.
- Build execution is forbidden unless explicitly required by the task or release policy.
- No verification tooling may be invented without ADR.

## 38. TypeScript Validation Law (Hard)
If TypeScript is used:
- All existing type-check scripts must pass.
- `any` and unsafe casts must be minimal and justified.
- Type suppression must be explicit and reviewed.

## 39. GraphQL & Contract Validation Law (Hard)
If GraphQL (or equivalent) exists:
- Schema must validate.
- Codegen must succeed if defined in repository.
- Breaking changes require versioning strategy and ADR.
- Contract drift is forbidden.

## 40. Test Integrity Law
Where automated tests exist:
- All defined test scripts must pass.
- Flaky tests are treated as failures until stabilized.

## 41. Behavioral Preservation Law
Refactoring must not change externally observable behavior unless explicitly intended.

## 42. Migration Safety Law
Migrations and risky changes must define:
- Safety checks
- Rollback or recovery strategy when risk exists
- Clear success criteria

## 43. Release Safety Law
Production-impacting changes must ensure:
- Observability is in place
- Failure handling is defined
- Rollback plan exists when risk exists

Unsafe releases are prohibited.

---

# XIV. ESCALATION PROTOCOL

## 44. Ambiguity Escalation Law (Hard)
If architectural certainty is below high confidence:
- Escalation is mandatory.
- Assumptions are prohibited.
- Structural invention is forbidden.

---

# XV. AMENDMENT

## 45. Constitutional Amendment Law
Amendments require:
1. Architect approval
2. Impact analysis
3. Migration plan
4. Documentation update
5. Version increment
6. Team notification

The Constitution evolves intentionally.
