# Requesters and Orchestration Pattern

## 1) Why This Pattern Exists

Ejtmaa follows the requester orchestration style:
- Requesters are the business execution boundary.
- Controllers are thin transport adapters.
- Decorators define routing + actor authorization intent.
- Validation is explicit and colocated with use case methods.

This keeps behavior testable and prevents business logic from leaking into routes.

## 2) Registration and Dispatch Pipeline

### Auto-Registration

`backend/src/app/orchestrator/Orchestrator.ts`:
- Scans requester classes.
- Uses `@requester("ident")` and `@sub([...platforms], [...actors])`.
- Builds runtime map: `RegisteredRequesters[platform][actor][ident]`.
- Generates `requesters.{platform}.ts` type maps.

### Runtime Dispatch

`ControllerBase.requesterHandle(platform, actor)`:
1. Reads `:requester` and `:sub` from route params.
2. Resolves current user actor from auth middleware.
3. Verifies requester/sub exists in `RegisteredRequesters[platform]` where **`platform` is `website` or `cpanel`**.
4. Creates requester root instance using `Requester.AsRoot(context)`.
5. Executes requested sub with `body.values`.

## 3) Requester Method Structure (Canonical)

Each requester method should keep this sequence:
1. **Validate** input + facts using `withJoiFirewall`.
2. **Normalize** validated output into strict `outProps`.
3. **Execute** domain mutation/read.
4. **Respond** using `SimpleJsonBuilder`.

Typical pattern:
- `const {throwIfNotValid, startTransaction} = this.context`
- `validationRes = withJoiFirewall(...).asyncValidate(body.values, ...)`
- `if (validationRes.errors) throw throwIfNotValid(...)`
- perform DB ops
- `result.addMainMessages(...)`
- `return result.done()`

## 4) Facts Validation and Actor Gating

Shared rules in `backend/src/app/validation/joi_rules.ts`:
- `isByVisitorFacts`
- `isByCustomerFacts`
- `isByCustomerFacts`
- `isByCustomerFacts`
- Model-level validators (`isCustomer`, `isSupervisor`, etc.)

Use facts as second authorization layer:
- Layer 1: route + `@sub` mapping.
- Layer 2: facts schema (`by.customer`, `by.customer`, ...).
- Layer 3 (when needed): entity ability (`model.can(...)`).

## 5) Transaction Pattern

Execution context comes from `ControllerBase.exe()`:
- `startTransaction()` creates/joins managed transaction.
- Auto-commit on success.
- Auto-rollback on thrown error.

Rules:
- Use one requester-level transaction for related writes.
- Pass `{transaction}` to all DB mutations in that use case.
- Prefer post-commit side effects (`afterCommit`) for external calls (mailer/notify/integration).
- Requesters must emit notification side effects through typed `notify(...)` events after commit; the event handler owns `Notification` row creation and broadcaster fanout. Do not create notification rows directly from requester methods.

## 6) Current Requester Inventory

Registered requesters (decorated):

| Requester | Ident | Platform / actor | Subs |
|---|---|---|---|
| `AuthRequester` | `auth` | `website` / visitor | `registerCustomer`, `login`, `resetPassword` |
| `AuthRequester` | `auth` | `cpanel` / visitor | `supervisorLogin` |
| `CustomerRequester` | `customer` | `website` / customer | customer profile subs |
| `CustomerRequester` | `customer` | `cpanel` / supervisor | `read`, `update` |
| `NotificationRequester` | `notification` | `website` / customer | `deleteAll` |
| `SupervisorRequester` | `supervisor` | `cpanel` / supervisor | `read`, `update` |
| `PlatformSettingsRequester` | `platform_settings` | `cpanel` / supervisor | `read`, `update` |
| `WebsiteSettingsRequester` | `website_settings` | `cpanel` / supervisor | `read`, `update` |

Website registration rule: customer/visitor HTTP methods on `/website` use `@sub(["website"], [...])`.

Runtime-generated requester maps:

- `backend/requesters.website.ts`
- `backend/requesters.cpanel.ts`

## 7) Route-Level Contract

Requester form endpoint convention:

| Mount | Actor | Path |
|---|---|---|
| `/website` | visitor | `POST /website/forms/requester/:requester/:sub` |
| `/website` | customer | `POST /website/forms/customer/requester/:requester/:sub` |
| `/cpanel` | visitor (login) or supervisor | `POST /cpanel/forms/requester/:requester/:sub` |

Examples:
- `POST /website/forms/requester/auth/login`
- `POST /website/forms/customer/requester/customer/updateSettings`
- `POST /cpanel/forms/requester/auth/supervisorLogin`
- `POST /cpanel/forms/requester/supervisor/update`

The controller should only route; business behavior belongs in requester methods.

## 8) Common Extension Rules

When creating a new requester:
1. Create file under `backend/src/app/orchestrator/requesters`.
2. Add `@requester("ident")` on class.
3. Add `@sub(...)` for every public use-case method.
4. Use facts schema from `joi_rules.ts`.
5. Use transaction for writes.
6. Return `SJsonResult`.

When adding a new actor-guarded method:
- Add method-level `@sub` with minimum required actor.
- Validate facts for that actor (`isByXFacts`).
- Never rely on route-level guard alone.

### Enum Input Validation Rule (Requester Layer)

For enum-like business fields in requesters:
1. Prefer `joi.select(...)` over raw `joi.string().valid(...)`.
2. Accept select-object payloads (`{value,label}`) when UI sends them.
3. Use `raw: true` when requester execution expects the persisted enum value.
4. Keep TypeScript contracts aligned with model-exported enum type (single source of truth).

`withJoiFirewall` note:
- Presence is `required` by default in engine configuration.
- Do not add `.required()` redundantly unless intentionally overriding behavior.

### `outPropsType` And `Modify`

`outPropsType` must describe the validated output shape. Use `Modify<Props, {...}>` only when validation changes the type of an existing payload field.

Correct uses:
- `avatar_file: FileObject | null` becomes `avatar_file: string | null`.
- select payload becomes a persisted enum value.

Do not include unchanged fields in `Modify`. If `name` enters validation as `string` and leaves validation as `string`, it stays in `Props`.

When validation only adds resolved values, use an intersection:

```ts
outPropsType: {} as CreateProps & {
    customer: CustomerModel
}
```

When validation changes one submitted field and adds resolved values, combine `Modify` with an intersection:

```ts
outPropsType: {} as Modify<UpdateProps, {
    email: string
}> & {
    customer: CustomerModel
}
```

## 9) Anti-Patterns to Avoid

- Manual requester instantiation in controllers.
- Missing decorators (method becomes unreachable via dispatcher).
- Writing DB mutations without transaction in a multi-step flow.
- Doing heavy logic in controllers.
- Skipping facts validation because route is authenticated.
- Treating enum-like inputs as free text when select-based contract exists.
- Repeating unchanged fields inside `Modify<Props, {...}>`.
- Chaining Joi `.optional()` / `.allow(null)` on Opt-backed `joi_rules.ts` helpers instead of passing the helper's `optional: true` argument through to `Model.Opt`.

## 10) Whitespace and Normalization Law (Mandatory)

This section is a hard policy for requester input normalization and Joi contracts.

### 10.1 Human and business text fields

Examples: `name`, `display_name`, display titles, labels, free-text notes.

Rules:
- Keep internal spaces between words.
- Use `joi.string().trim()` for edge-space cleanup.
- Do not use `prepareCleanString(..., { cleanWhiteSpaces: true })` for these fields.
- Do not collapse `"Abdullah Al Harbi"` into `"abdullahalharbi"` in validation or persistence paths.

### 10.2 Machine-safe identifiers only

`cleanWhiteSpaces: true` is allowed only for fields where spaces are semantically invalid and should be removed for canonical matching/storage.

Allowed examples:
- `email`
- `mobile`

### 10.3 Joi contract style

Prefer clear Joi composition over complex custom callbacks when built-in rules are sufficient.

Required approach:
- Use `trim().min(...).required()` for text-length constraints.
- Use `.external(...)` only when asynchronous/DB checks are needed.
- When a value is normalized inside `.external(...)`, downstream uniqueness checks must use the normalized value, not the raw input.

### 10.4 MultiLang registration bridge rule

For single-input registration fields that are persisted as `MultiLangString`:
- client sends one trimmed string,
- requester maps it inline to `{ ar: value, en: value }`,
- no generic helper should be introduced for this copy step.

### 10.5 Model.Opt optional propagation (Mandatory)

`Model.Opt(joi, path, optional?)` owns absent-value behavior inside its `external` resolver:
- missing value + `optional !== true` -> `error("any.required")`
- missing value + `optional === true` -> pass through without lookup

Rules:
- Opt-backed `joi_rules.ts` helpers expose `optional?: boolean` and forward it to `Model().Opt`.
- When a chained `.external(...)` follows `Model().Opt`, start with `if (optional && !value) return;` before business checks.
- Call sites pass `true` when the field may be omitted/null, for example `isByCustomerFacts(joi, true)`.
- Forbidden: chaining `.optional()` or `.allow(null)` on an Opt-backed helper return value.
- Plain Joi fields (`joi.string()`, `isMultiLang(joi)`, etc.) still use Joi `.optional()` / `.allow(null)` normally.

Ejtmaa Opt helpers live in `backend/src/app/validation/joi_rules.ts` (`isByCustomerFacts`, `isBySupervisorFacts`, etc.).

## 11) Reference Alignment

Ejtmaa uses the requester orchestration style documented here.

## 12) Governance Traceability

| Path | Contract |
| --- | --- |
| `docs/platforms/backend/contracts/client-portal-http-website.md` | `/website` client-portal HTTP mount contract. |
| `docs/platforms/backend/contracts/http-and-requesters.md` | Global HTTP/requester envelope. |
| `docs/platforms/backend/contracts/account-settings-requesters.md` | Customer account settings requester contract. |
| `docs/platforms/backend/patterns/requesters-and-orchestration.md` | This requester/orchestration reference. |
