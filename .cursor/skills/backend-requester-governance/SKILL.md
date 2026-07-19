---
name: backend-requester-governance
description: Enforces the full Ejtmaa backend requester constitution. Use when creating, updating, fixing, or reviewing backend requesters, joi_rules, Ability, Able, can, @requester, @sub, requesterHandle, requester transactions, side effects, validation, or requester-owned authorization.
---

# Backend Requester Governance

## When to Use

Use this skill for any task that creates, updates, repairs, or reviews:

- `backend/src/app/orchestrator/requesters/**/*.ts`
- `backend/src/app/validation/joi_rules.ts`
- requester-facing `Ability`, `Able`, or `can` logic in `backend/src/app/orm/models/**/*.ts`
- requester controllers, `requesterHandle`, `@requester`, or `@sub`
- requester transactions, `afterCommit`, manual commit, notifications, email, or external side effects

## Instructions

1. Read Source Of Truth and locate the owning requester under `backend/src/app/orchestrator/requesters/`.
2. Wire `@requester` + `@sub`; keep controllers thin via `requesterHandle`.
3. Validate with `joi_rules.ts` helpers; use `outPropsType` / `Modify` correctly.
4. Authorize inside the requester with `actor.can(...)`; never skip because the route is authed.
5. Keep multi-step writes inside one transaction owner; make side effects idempotent.
6. Run `yarn type-check` in `backend/`.

## Source Of Truth

This skill is self-contained Ejtmaa governance. Do not mention, cite, compare against, or defer to external project names in requester work. Use only:

- this skill
- `.cursor/rules/backend-requesters-governance.mdc`
- `.cursor/rules/backend-requester-whitespace-normalization.mdc`
- nearby local requesters
- `backend/src/app/validation/joi_rules.ts`
- `backend/src/app/orm/models/Model.ts`
- directly related local models/controllers

If local code and this governance conflict, stop and ask. Do not silently choose a new convention.

## Requester Ownership

Requesters orchestrate one user-facing write/read action. They do not own reusable domain permission logic, reusable actor resolution, or controller routing.

Requester owns:

- selecting the requested sub action
- validating input for that action
- starting the transaction when needed
- calling model/domain operations in order
- calling `can(..., { throwMode: true, transaction })` when state-changing permission applies
- building `FormResult`
- choosing and documenting side-effect timing

Requester does not own:

- manual requester registration
- repeated actor/facts resolution
- repeated ownership/access validation
- reusable domain permission rules
- GraphQL/UI ability display contracts
- broad normalization policy

## File Shape

New requester files must use this shape:

```typescript
import RequesterBase, {SJsonResult} from "../RequesterBase";
import {FormBody, FormResult} from "../../types/FormsTypes";
import {SimpleJsonBuilder} from "@nodejs/render";
import {withJoiFirewall} from "../../helpers/ability-engine/AbilityEngine";
import {requester, sub} from "../Orchestrator";

export namespace ISupervisorRequester {
    export type ChangePasswordProps = {}
    export type ChangePasswordResult = {}

    @requester("supervisor")
    export class Requester extends RequesterBase {
        @sub(["cpanel"], ["supervisor"])
        async changePassword(
            body: FormBody<ChangePasswordProps>,
            result: SimpleJsonBuilder<FormResult<ChangePasswordResult>>
        ): Promise<SJsonResult> {
            // implementation
        }
    }
}

export default ISupervisorRequester.Requester;
```

Required file rules:

- The file path is `backend/src/app/orchestrator/requesters/<Name>Requester.ts`.
- The namespace is `I<Name>Requester`.
- The class name is exactly `Requester`.
- The class extends `RequesterBase`.
- The class has `@requester("ident")`.
- Each callable public method has `@sub(platforms, actors)`.
- The default export is `<Namespace>.Requester`.
- Controllers must not import and register the requester manually.
- If a requester has helper functions, place them after the class inside the namespace or as local private file helpers only when they are not reusable across requesters.

## Method Contract

- The method returns `Promise<SJsonResult>`.
- Use `body: FormBody<Props>`.
- Use `result: SimpleJsonBuilder<FormResult<Result>>` for data-returning methods.
- Use `result: SimpleJsonBuilder<FormResult>` for mutation-only methods.
- The method never returns raw data, model instances, booleans, or plain objects.
- Result data goes through `result.setData(...)`.
- Success messages go through `result.addMainMessages("SUCCESS", [await this.msg("...")])`.
- The final line is normally `return result.done()`.
- Use `sJson()` default only when local usage requires calling the method from code without passing a result builder.

## Required Method Flow

The method body must follow this order:

```typescript
const {throwIfNotValid, startTransaction} = this.context;

//validate
const validationRes = await withJoiFirewall(joi => {
    return joi.object({
        facts: isByActorFacts(joi),
        notification: Notification().Opt(joi, "notification")
    });
}, await this.joiOptions())
    .asyncValidate(body.values, {
        outPropsType: {} as Modify<ActionProps, {}> & {
            actor: ActorModel,
            notification: NotificationModel
        }
    });

if (validationRes.errors)
    throw throwIfNotValid(validationRes.errors);

const props = validationRes.outProps;
const actor = props.actor;
const notification = props.notification;

//execute
const transaction = await startTransaction();

await actor.can("NOTIFICATION", {
    sub: "deleteAll",
    transaction,
    throwMode: true
});

await Notification().destroy({ where: { customer_id: actor.get("id") }, transaction });

//finally
result.addMainMessages("SUCCESS", [await this.msg("SUCCESS_UPDATE")]);
return result.done();
```

Hard flow gates:

- No mutation before validation succeeds.
- No side effect before validation succeeds.
- No `can(... throwMode: true ...)` after mutation when it should guard the mutation.
- No transaction start for pure read methods unless a transaction is needed for consistency.
- No broad `try/catch` that swallows validation, permission, or transaction errors.
- No direct `this.res`, `this.req`, or controller access from requester methods.

## Validation Governance

Validation always uses:

```typescript
withJoiFirewall(joi => {
    return joi.object({...});
}, await this.joiOptions())
    .asyncValidate(body.values, {outPropsType: ...});
```

Validation must include `facts` for every routed requester method. `facts` is actor context injected by requester orchestration. It is not trusted client data.

Use this decision rule:

- One actor allowed: `facts: isByActorFacts(joi)`.
- Multiple actors allowed: `facts: joi.alt(isByActor1Facts(joi), isByActor2Facts(joi))`.
- Visitor-only action: `facts: isByVisitorFacts(joi)`.
- Existing model reference: `Model().Opt(joi, "prop")` or a `joi_rules.ts` helper based on it.
- Repeated ownership/access rule: `joi_rules.ts` helper.
- One-off field shape rule: inline Joi in the requester.
- Async uniqueness or lookup rule specific to this action: inline `.external(...)`.
- Reusable domain permission rule: model `can(...)`, not Joi.

## `joi_rules.ts` Constitution

Use `backend/src/app/validation/joi_rules.ts` for reusable validation contracts.

Actor facts helper shape:

```typescript
export const isByCustomerFacts = (joi: JoiFirewallRoot, withUser?: boolean) => joi.object({
    by: joi.object({
        customer: Customer().Opt(joi, "customer")
            .external(async (customer: CustomerModel, helpers) => {
                const {danger, set} = joi.smartHelpers(helpers, true);

                if (withUser) {
                    const user = await customer.getUser();
                    if (!user)
                        return danger("NOT_PERMIT");
                    set("user", user);
                }

                return customer;
            })
    })
});
```

Supervisor facts helper shape:

```typescript
export const isBySupervisorFacts = (joi: JoiFirewallRoot, withUser?: boolean) => joi.object({
    by: joi.object({
        supervisor: Supervisor().Opt(joi, "supervisor")
            .external(async (supervisor: SupervisorModel, helpers) => {
                const {danger, set} = joi.smartHelpers(helpers, true);

                if (withUser) {
                    const user = await supervisor.getUser();
                    if (!user)
                        return danger("NOT_PERMIT");
                    set("user", user);
                }

                return supervisor;
            })
    })
});
```

Ownership helper shape:

```typescript
export const isOwnedNotification = (joi: JoiFirewallRoot, optional?: boolean) => Notification().Opt(joi, "notification", optional)
    .external((notification: NotificationModel, helpers) => {
        const {danger, get} = joi.smartHelpers(helpers, true);
        const customer = get<CustomerModel>("customer");

        if (customer && notification.get("customer_id") !== customer.get("id"))
            return danger("NOT_PERMIT");

        return notification;
    });
```

Rules:

- `set("name", value)` adds resolved values to `outProps`.
- `get("name")` reads values resolved earlier in the same validation pipeline.
- `withUser` is opt-in. Use it only when the user relation is required.
- Use `danger("NOT_PERMIT")` for authorization/domain denial.
- Use `danger("404")` when a referenced entity is not found.
- Use `error("field.code")` for field-specific validation failures.
- Do not put state-transition permission logic in `joi_rules.ts`; put that in `can`.
- Opt-backed helpers accept an optional second argument (`optional?: boolean`) and forward it to `Model().Opt(joi, path, optional)`. When a field may be absent, pass `true`. Do not chain `.optional()` / `.allow(null)` on the helper instead of forwarding `optional: true`.
- Discriminator-driven branch refs prefer `joi.when("targetType", { is, then, otherwise: joi.forbidden() })` inside one flat requester-owned `joi.object`; derive ids from resolved models in the requester instead of `concat` + `set("targetId")`.
- When a helper chains `.external(...)` after `Model().Opt`, guard absent values first: `if (optional && !model) return;`

## Existing Record References

Existing records must use `Model().Opt(joi, "prop")` or a helper built on it. Do not validate an id with `joi.number()` and then call `Model.find(...)` inside the requester. `Model().Opt` accepts id/select/model-like input, resolves the model, handles not found, and places the model instance in `outProps`.

When the field may be omitted or null, call `Model().Opt(joi, "prop", true)` or `isSomeModelRef(joi, true)`. Inside `Model.Opt`, absent values skip `any.required` only when that third argument is `true`; outer Joi `.optional()` does not set that flag.

## Select And Enum Inputs

Use `joi.select({validValues: [...]})` for enum/select payloads when the field accepts select-style frontend values.

Use `outPropsType` to reflect the final persisted type:

```typescript
.asyncValidate(body.values, {
    outPropsType: {} as Modify<Props, {
        status: CustomerStatus
    }>
});
```

Do not manually extract `SelectOption.value` in requester code unless the existing local contract requires it and the reason is clear.

### Read Hydrate (FormChoice / EntityPicker)

When `read` returns values for website choice tiles or entity pickers, return SelectOption shapes — do not leave bare enum strings or bare FK ids.

- Enums: `await this.toEnumForSelect(value, enumKey)` (`RequesterBase`).
- Related entities: per-model `forSelect(lang)` on the related ORM model; call from `read`.
- `ReadResult` types those fields as `SelectOption` / `Nullable<SelectOption>`.
- Write path stays `joi.select` — do not require SelectOption-only writes.

Authority: `docs/platforms/backend/patterns/requester-read-select-hydrate.md`. Skill: `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`. Rule: `.cursor/rules/requester-read-select-hydrate.mdc`.

### Type-Conditional Unused Fields

When columns apply only for some `type` (or similar discriminator) values:

- Unused branches: `joi.any().optional().allow(null, "").strip()`.
- Persist flat: `props.field ?? null`.
- Do **not** rebuild per-type `attrsForType` maps whose only job is nulling.
- Prefer `.strip()` over `forbidden()` when the UI may still send stale keys.

Authority: `.cursor/rules/requester-type-conditional-strip.mdc`. Canonical: `MessageTemplateRequester`, `MessageChannelRequester`.

## `outPropsType` Rules

`outPropsType` is part of the contract. It must describe what validation returns, not what the client sent.

Use `Modify<Props, {...}>` when:

- id/select becomes a model instance
- file object becomes a stored filename
- select/enum object becomes an enum string
- nullable values are normalized

Do not put unchanged fields in `Modify<Props, {...}>`. It is a type replacement tool, not a restatement of validated fields. If `name` and `description` enter validation as `string` and leave as `string`, keep them in `Props`.

When validation adds values that were not in the client payload, append them with an intersection:

```typescript
outPropsType: {} as CreateProps & {
    customer: CustomerModel
}
```

When validation changes one field and also adds helper values, combine both shapes:

```typescript
outPropsType: {} as Modify<UpdateProps, {
    email: string
}> & {
    customer: CustomerModel
}
```

Incorrect:

```typescript
outPropsType: {} as Modify<UpdateProps, {
    email: string,
    name: string,
    description: string
}> & {
    customer: CustomerModel
}
```

Correct:

```typescript
outPropsType: {} as Modify<UpdateProps, {
    avatar_file: Nullable<string>
}> & {
    customer: CustomerModel,
    user: UserModel
}
```

Also correct when no existing field type changes:

```typescript
outPropsType: {} as CreateProps & {
    customer: CustomerModel
}
```

## Authorization Constitution

Requester authorization has three layers. Do not collapse them into one.

1. Route layer: `@sub(platforms, actors)`.
2. Context layer: `facts` validation in Joi.
3. Domain layer: `model.can(...)` before state changes.

Use `can` when the answer depends on domain ownership or entity state:

- Can this actor update this record?
- Can this status be changed now?
- Can this record be canceled, deleted, published, verified, replied to, or closed?
- Is there an existing pending/confirmed/conflicting record?
- Is the requested state transition legal?

Use Joi when the answer is input validity:

- Is the field present?
- Is the field a valid email/mobile/date/file?
- Does the referenced model exist?
- Does a reusable ownership/access helper need to reject the reference before execution?

If a check is both reusable and permission-like, default to `can` unless it only validates a reference in the payload.

## `Ability`, `Able`, And `can`

Model permission contract:

```typescript
type Ability = {
    NOTIFICATION: {
        sub: "deleteAll"
    }
}

export default class CustomerModel extends Model<Attrs, CreationAttrs, Ability> {
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
}
```

Rules:

- `Ability` belongs in the model that owns permission.
- The requester never defines `Ability`.
- `Ability` groups permissions by domain noun (`NOTIFICATION`, `LOGIN`, etc.).
- Each domain group has a `sub` union.
- Optional target props allow both general checks and target-specific checks.
- `can` must return `this.iCan(...)`.
- Throw stable message codes inside `can`; do not return booleans.
- `Able` success is `"CAN"`.
- Without `throwMode`, failure returns a string code or visual object.
- With `throwMode`, failure throws and must stop the requester.
- `visualMode` is for read/GraphQL/UI ability surfaces, not normal mutation flow.
- Any query inside `can` that belongs to the active mutation receives the same transaction passed by the requester.

Mutation usage:

```typescript
await actor.can("NOTIFICATION", {
    sub: "deleteAll",
    transaction,
    throwMode: true
});
```

## Transaction Constitution

Default:

- `ControllerBase.exe()` owns final commit/rollback.
- Requester starts a transaction only when needed.
- All DB writes use the same transaction.
- Any `can` query that participates in the operation receives the same transaction.
- Do not create nested independent transactions for one requester operation.

Use:

```typescript
const {throwIfNotValid, startTransaction} = this.context;
const transaction = await startTransaction();
```

Manual close:

- Manual `transaction.commit()` or rollback inside requester is forbidden by default.
- It is allowed only when there is a clear logical reason.
- The reason must be stated in the plan or explanation.
- Explicit user approval is required before implementation.
- After manual commit, later failures cannot roll back earlier DB writes; the requester must account for that.

## Side-Effect Constitution

Side effects include email, notifications, external calls, webhooks, filesystem effects, and any action outside the local DB transaction.

Decision rule:

- If the requester may be reused by another requester or share a transaction, prefer `transaction.afterCommit(...)`.
- If the side effect must happen after guaranteed DB commit and should not be rolled back, use `afterCommit` or approved manual commit.
- If the side effect must be awaited before returning, explain why and get approval when it requires manual commit.
- Never send side effects before validation and permission checks.
- Never send side effects inside a transaction without thinking through rollback and duplication.

Use `transaction.afterCommit(() => { ... })` for post-commit side effects. Use `await transaction.commit()` before a side effect only when explicitly approved.

## Text Normalization

Always apply `.cursor/rules/backend-requester-whitespace-normalization.mdc`.

Summary:

- Human/business text uses `joi.string().trim()` and preserves internal spaces.
- Machine identifiers such as email/mobile may use strict cleanup when spaces are invalid.
- If `.external(...)` normalizes a value, uniqueness checks must use the normalized value.
- Single input persisted as `MultiLangString` maps inline to `{ar: value, en: value}` unless local code already has a more specific contract.

## Anti-Patterns

Reject these patterns:

- requester outside `backend/src/app/orchestrator/requesters/`
- requester class without `@requester`
- callable method without `@sub`
- manual requester maps in controllers
- raw method return values
- mutation before validation
- permission check after mutation
- raw `joi.number()` for existing record references followed by requester-owned `find`
- repeated ownership logic copied across requesters
- reusable permission logic implemented inline in requester
- `can` returning boolean
- `Able` treated as boolean
- `visualMode` used for mutation enforcement
- manual transaction close without explicit approval
- side effects hidden inside transaction
- collapsed internal spaces for human/business text
- external project names in requester guidance, comments, docs, or final explanation

## Review Checklist

Before finishing requester work, verify:

- file path and namespace match requester name
- `@requester` ident matches route/API intent
- every callable public method has `@sub`
- method signature returns `Promise<SJsonResult>`
- validation uses `withJoiFirewall` and `this.joiOptions()`
- `facts` matches `@sub` actors
- `outPropsType` matches resolved values
- model references use `Model().Opt` or `joi_rules.ts` helpers
- reusable validation belongs in `joi_rules.ts`
- reusable domain permission belongs in model `can`
- state-changing requesters call `can(... throwMode: true ...)` where applicable
- transaction is started only when needed
- all DB writes and related `can` queries share the transaction
- manual commit is absent unless approved
- side effects are after validation, after permission, and timed deliberately
- result builder is used consistently
- whitespace normalization policy is satisfied
- no external project name appears in new guidance or explanation

## Verification

- For requester/model code changes, run only existing backend package scripts relevant to the touched area.
- For `.cursor/**`-only edits, read the edited files, check trigger terms, and run lint diagnostics if available.
- Search edited guidance for external project names and remove them.
