---
name: backend-able-can-gql-contract
description: Defines Ejtmaa Ability, Able, can, iCan, throwMode, visualMode, and GraphQL ability exposure contracts. Use when adding or fixing model permissions, requester permission checks, GQL ability fields, MeBridge abilities, loadExtra ability outputs, or UI-facing can/cannot reasons.
---

# Backend Able Can GQL Contract

## When to Use

Use this skill when work touches:

- `type Ability` in `backend/src/app/orm/models/**/*.ts`
- `can(...)`, `iCan(...)`, `Able`, `throwMode`, or `visualMode`
- requester calls to `actor.can(...)`
- GraphQL ability fields, `MeBridge`, `static extras`, or `loadExtra`
- SDL fields that expose whether an actor can perform an action
- UI-facing permission reasons or translated ability descriptions

## Instructions

1. Read Source Of Truth and confirm which model owns the permission (`Customer`, `Supervisor`, `User`).
2. Add or update `Ability` on the owner model; implement `can(...)` with `iCan`.
3. For UI-facing gates, expose `_Ability` fields on `_Me` or bounded entity extras via bridge `loadExtra`.
4. Keep requester writes guarded even when GQL extras return `CAN`.
5. Regenerate GQL types and mirror SDL to consuming frontends when SDL changes.
6. Run `yarn type-check` in `backend/`.

## Source Of Truth

This is a self-contained Ejtmaa contract. Do not mention, cite, compare against, or defer to external project names.

Read before editing:

- `backend/src/app/orm/models/Model.ts`
- the actor model that owns the permission
- requester(s) that enforce the same action
- related `MeBridge` or entity bridge
- related GraphQL definition file under `backend/src/app/gql/definitions/`
- `.cursor/rules/backend-able-can-gql-contract.mdc`
- `.cursor/rules/backend-requesters-governance.mdc`
- `.cursor/rules/gql-schemas-bridges-general.mdc`

If local code and this contract conflict, stop and ask.

## Purpose

`can(...)` is the domain permission contract. It answers: "Can this actor perform this domain action, optionally on this target, under the current state?"

`Able` is the result shape of that permission contract. It is intentionally not boolean:

- mutation paths need failure to throw and stop execution
- read/UI/GQL paths need a stable value and optional translated description
- the same permission logic must serve both paths without duplicating rules

GraphQL uses `Able` to expose capability and denial reasons to clients so the UI can hide, disable, explain, or refresh actions without re-implementing backend policy.

## Ownership Decision

Put logic in `can(...)` when it is reusable domain permission:

- ownership checks for performing an action
- status transitions
- whether a record can be deleted, canceled, submitted, verified, replied to, published, closed, or updated
- conflict checks such as existing pending/confirmed records
- limits that determine whether an actor may perform the action

Keep logic in Joi/requester when it is input orchestration:

- field shape and requiredness
- resolving a payload reference into a model instance
- one-off uniqueness checks specific to the submitted values
- sequencing writes and side effects

Keep logic in GraphQL bridge when it is read exposure:

- selecting which ability fields were requested
- calling `can(... visualMode ...)`
- mapping `Able` into the SDL output shape

## Model Ability Contract

Define `Ability` beside the model that owns permission:

```typescript
type Ability = {
    NOTIFICATION: {
        sub: "deleteAll"
    }
}
```

Wire it into the model:

```typescript
export default class ActorModel extends Model<Attrs, CreationAttrs, Ability> {
}
```

Rules:

- `Ability` belongs to the actor/owner model, not the requester and not GraphQL.
- The first key is a domain noun such as `NOTIFICATION`, `CUSTOMER`, or `AUTH`.
- `sub` is mandatory and must be a union of allowed actions.
- Target props are optional only when the action supports general ability checks.
- Target prop types may accept model instance or id-like values when `can` resolves the target itself.
- Keep the type narrow; do not use `{ [key: string]: any }` for new permissions.

## `can(...)` Implementation Contract

Implement `can` with `iCan`:

```typescript
async can<To extends keyof Ability, CanProps extends CanPropsBase<Ability[To]>>(
    to: To,
    props: CanProps
): Promise<Able> {
    return this.iCan(async () => {
        switch (to) {
            case "NOTIFICATION": {
                const _props = props as Ability["NOTIFICATION"];

                switch (_props.sub) {
                    case "deleteAll": {
                        break;
                    }
                }

                break;
            }
        }
    }, props);
}
```

Rules:

- Always return `this.iCan(...)`.
- Throw stable message keys inside the test body.
- Do not return booleans.
- Do not return `"CAN"` manually except in the base/default implementation.
- Resolve target models when the permission accepts id-like values.
- Pass `transaction` into every query that participates in an active write operation.
- A missing optional target means a general ability check; only allow this when meaningful.
- Unknown `to` or `sub` must not silently grant a real permission. Add explicit cases for new permissions.

## `Able` Semantics

`Able` is:

```typescript
"CAN" | string | { value: "CAN" | "CANNOT" | string, description?: string }
```

Meaning:

- `"CAN"` means the action is allowed.
- a string means a denial or failure code such as `"NOT_PERMIT"`.
- an object is a visual result for UI/GQL with `value` and translated `description`.

Do not use `if (await actor.can(...))` because a denial string is truthy. Compare explicitly or use `throwMode`.

## Mutation Usage

Mutation/requester paths enforce permissions with `throwMode: true`:

```typescript
await actor.can("NOTIFICATION", {
    sub: "deleteAll",
    transaction,
    throwMode: true
});
```

Rules:

- Call before the mutation it protects.
- Pass the active transaction when the requester has one.
- Do not use `visualMode` for mutation enforcement.
- Do not duplicate the same domain rule inline after calling `can`.

## Read And GraphQL Usage

Read/UI/GQL paths use `visualMode` when the client needs a displayable reason:

```typescript
const able = await actor.can("NOTIFICATION", {
    sub: "deleteAll",
    visualMode: {lang: this.context.lang()}
});
```

Use GQL exposure when:

- the frontend must decide whether to show/disable an action
- the frontend needs a backend-owned reason why the action is unavailable
- the ability depends on ownership, current status, related records, limits, or time
- duplicating the rule in frontend would create drift

Do not expose a GQL ability when:

- the action is always allowed for the current actor
- the UI does not need it
- the rule is simple input validation handled only at submit time
- exposing it would require unbounded per-row `can` calls on large lists

## GQL Shape

Prefer an ability object type with `value` and `description`:

```graphql
type _Ability {
    value: String!
    description: String
}

type _Me {
    canDeleteNotifications: _Ability
}
```

Rules:

- Reuse the shared `_Ability` type from `base.graphql`.
- Ability fields on `_Me` answer "what can the current actor do?" (for example `canDeleteNotifications`).
- Entity-specific ability fields may be entity extras only when they are root-one, bounded, and needed beside that entity.
- Avoid ability extras on unbounded root-many lists unless the query is bounded and the performance cost is accepted.
- Keep SDL, generated GQL types, bridge types, and website mirrors in sync when SDL changes.

## MeBridge Pattern

Expose `me.abilities` through `MeBridge` when ability output is about the current actor:

```typescript
static extras = ["abilities"];

async loadExtra(option: LoadExtraOptions<ActorModel, GetOneParent>): Promise<any> {
    const {extra, model} = option;
    if (extra !== "abilities") return;

    const actor = this.context.actor;
    if (!actor) throw this.throw("NOT_PERMIT");

    return {
        canDeleteNotifications: await actor.can("NOTIFICATION", {
            sub: "deleteAll",
            visualMode: {lang: this.context.lang()}
        })
    };
}
```

## Notification bridge ability pattern

Use notification bridge extras for per-record ability when bounded and necessary:

```typescript
static extras = ["canDelete"];

async loadExtra(option: LoadExtraOptions<NotificationModel, Parent>): Promise<any> {
    const {extra, model} = option;
    if (extra === "canDelete") {
        const actor = this.context.actor;
        if (!actor) throw this.throw("NOT_PERMIT");

        return actor.can("NOTIFICATION", {
            sub: "deleteAll",
            visualMode: {lang: this.context.lang()}
        });
    }
}
```

Rules:

- Use notification extras for one-model detail screens or small bounded lists.
- Prefer `me.abilities` for action-level capabilities that are not naturally a field on the entity.
- Avoid N+1 ability calculations on broad lists.
- Ensure `getRequiredOrmAttrs(...)` loads fields needed by `can`, or have `can` resolve a complete model safely.

## GraphQL Performance And Safety

- Ability fields must not make unbounded per-row queries.
- If `can` needs target fields not guaranteed loaded by the bridge, either load required attrs or resolve the full target in `can`.
- Principal guards remain mandatory; unauthenticated ability output must return a stable denial or throw based on the schema contract.
- Do not leak another actor's ownership by returning `"CAN"`/denial details for records outside the role scope.
- Keep ability outputs deterministic and side-effect free.

## Requester And GQL Consistency

The same `can` rule must be used by:

- requester mutation enforcement with `throwMode`
- GQL/UI ability exposure with `visualMode`

If GQL says an action is allowed but requester denies it, the task is incomplete. Fix the shared `can` contract or the GQL args/target used to call it.

## Review Checklist

- `Ability` is defined on the owning actor model.
- `Model<Attrs, CreationAttrs, Ability>` is wired.
- `can` returns `this.iCan(...)`.
- denial uses stable message keys.
- requester mutations call `can(... throwMode: true ...)`.
- GQL ability surfaces call `can(... visualMode ...)`.
- `Able` is never treated as boolean.
- GQL ability fields are bounded and side-effect free.
- SDL, generated types, bridges, and app mirrors are synchronized when SDL changes.
- No external project names appear in guidance, comments, docs, or final explanation.

## Verification

- For model/requester/GQL code changes, run only existing scripts from the touched package.
- If GraphQL SDL changes, run the existing type generation/sync process used by this repository.
- For `.cursor/**`-only edits, read the changed files, search for external project names, and run lint diagnostics if available.
