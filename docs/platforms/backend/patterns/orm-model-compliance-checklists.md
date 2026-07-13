# ORM Model Compliance Checklists

Use this document before and after any model generation or modification.

## A) Pre-Implementation Checklist

- [ ] Model classification selected: actor or non-actor.
- [ ] Domain boundaries and ownership semantics identified.
- [ ] GQL exposure status confirmed (not exposed / root query / nested relation / extras).
- [ ] Required relation paths listed (requester + bridge consumers).
- [ ] Required indexes/scopes listed.
- [ ] Ability contract required? decision is explicit.

## B) Attributes Checklist

- [ ] `type Attrs` includes all persisted fields used by logic.
- [ ] Virtual fields used by consumers are represented in `Attrs`.
- [ ] `attributes()` reflects field nullability/default behavior.
- [ ] Enum-like fields include consistent metadata where needed.
- [ ] JSON fields have typed shape and proper metadata.
- [ ] No unused or dead attributes added.

## C) Options Checklist

- [ ] `modelName` and `tableName` are set and consistent.
- [ ] Required unique/performance indexes are present.
- [ ] Scopes are defined only for real query use-cases.
- [ ] No incompatible naming conventions introduced.

## D) Relations Checklist

- [ ] All required domain relations are defined in `boot()`.
- [ ] FK direction is correct (`belongsTo` vs `has*`).
- [ ] Polymorphic relations use explicit discriminator strategy.
- [ ] `constraints: false` appears only when polymorphism needs it.
- [ ] Relation aliases and mixins align with consumer calls.

## E) Ability and Can Checklist (Actor Models)

- [ ] `Ability` type defines operation branches and props shape.
- [ ] `can()` uses `iCan()` wrapper.
- [ ] Every branch validates ownership/facts.
- [ ] Denial paths return stable message keys.
- [ ] Requester layer calls `can()` before write effects.
- [ ] Actor-owner parity reviewed (for example staff owners like supervisor are explicitly intentional if no `can()` override).

## F) Helper Functions Checklist

- [ ] Helpers are deterministic and typed.
- [ ] Helpers avoid hidden writes or global side effects.
- [ ] Helpers used by bridges do not cause avoidable N+1.
- [ ] Helper names express domain behavior clearly.

## G) ORM ↔ GQL Checklist

- [ ] All attrs used in bridge sort/filter are loaded (`requiredOrmAttrs` or include attrs).
- [ ] Bridge extras relying on model fields have guaranteed field availability.
- [ ] Union/polymorphic discriminators are loaded for bridge resolution.
- [ ] Root-many queries remain bounded and ordered.
- [ ] Deep include policy and depth caps remain active.

## H) Security Checklist

- [ ] No resolver-only security for mutation critical paths.
- [ ] Owner-sensitive operations validated by facts + ability checks.
- [ ] Error keys are consistent with localization and caller handling.
- [ ] No hardcoded principal IDs introduced.

## I) Final Acceptance Checklist

- [ ] `orm-model-authoring-standard.md` rules satisfied.
- [ ] Nearest worked example mapped and explained.
- [ ] Anti-pattern scan completed with no violations.
- [ ] Change is consistent with `models-owners-abilities-security.md`.
