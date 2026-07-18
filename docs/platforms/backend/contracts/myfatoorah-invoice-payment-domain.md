# MyFatoorah Invoice & Subscription Payment Domain (Current)

## 1) Scope

Shipped Ejtmaa payment / gateway-session surface:

- ORM `MyFatoorahInvoice` (gateway payment session owned by Customer),
- customer requester `subscription.subscribe` → PENDING invoice + MyFatoorah payment URL,
- customer GQL reads: `subscriptionPaymentMethods(planId, billingPeriod)`, `_Me.canSubscribe(planId)`,
- HTTP `/external` finalize callback → invoice status + entitlement create via `Subscription.subscribe`,
- website type mirrors only (SDL + gql-types + `requesters.website.ts`),
- localization for `myFatoorahInvoiceStatus`.

Out of scope (not shipped):

- customer or supervisor GraphQL roots / bridges for invoice rows (`_MyFatoorahInvoice` does not exist),
- website UI (pages, hooks, adapters, forms, payment-return screens),
- cpanel / supervisor invoice or payment surfaces,
- stale `PENDING` invoice reconcile / expire scheduler,
- renew-specific requester beyond `subscribe`,
- `Order` / `OrderPayment` layers,
- reintroducing `PENDING` / `CANCELED` on `Subscription` (checkout stays on the invoice).

Related contracts:

- `subscription-domain.md` — entitlement row + `Subscription.subscribe` static
- `plan-domain.md` — catalog prices used for amount + method listing
- `external-http-mount-and-myfatoorah-callbacks.md` — `/external` mount + redirect

## 2) Domain purpose

```text
Plan (catalog)
  ↑ snapshot
Subscription (customer entitlement; only after paid success)
  ↑ soft link via related_data.subscription_id (no ORM FK)
MyFatoorahInvoice (gateway session; money truth = amount)
```

| Layer | Owns |
|---|---|
| `Plan` | Catalog list prices / limits |
| `MyFatoorahInvoice` | Gateway session, charged `amount`, soft intent in `related_data`, MF refs |
| `Subscription` | Paid entitlement window + commercial snapshot |

Rules:

- Owner is **Customer** only (`customer_id` real FK) — not Organization, not polymorphic owner.
- No subscription row before successful payment finalize.
- Shared money lives on invoice column `amount` only — never inside `related_data` arms; no `payed_amount`.
- `related_data` is a **discriminated union** on `pay_for`. Extend by adding new arms; never flatten into one shared field bag.

## 3) ORM: `MyFatoorahInvoice`

File: `backend/src/app/orm/models/MyFatoorahInvoice.ts`

Classification: **non-actor** (no `Ability`, no `can()`).

Persistence:

- `modelName`: `myFatoorahInvoice`
- `tableName`: `my_fatoorah_invoices`
- Factory: `export const MyFatoorahInvoice = () => ormModel(m => m.MyFatoorahInvoice) …`

Local registry (gitignored): `backend/.types/models.ts` must include `"MyFatoorahInvoice"`.

### 3.1 Columns

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | BIGINT PK | no | auto-increment |
| `customer_id` | BIGINT | no | payer Customer |
| `private_key` | STRING(191) | no | unique callback key (rewritten from MF helper after URL create) |
| `amount` | DECIMAL(18, 2) | no | SAR charged for this session |
| `related_data` | JSONB | no | discriminated soft intent (`isTypedObject`) |
| `status` | STRING(191) | no | `myFatoorahInvoiceStatus`, default `PENDING` |
| `mf_invoice_id` | BIGINT | yes | unique when set |
| `mf_method_id` | BIGINT | yes | selected gateway method |
| `mf_payment_url` | TEXT | yes | hosted payment URL |
| `mf_payment_id` | STRING(191) | yes | from callback query when present |

### 3.2 Statuses

| Value | Meaning |
|---|---|
| `PENDING` | Session open / awaiting gateway outcome |
| `COMPLETED` | Paid; entitlement create attempted for subscription arm |
| `FAILED` | Failed / unpaid error path (from `PENDING` only) |
| `CANCELED` | Gateway canceled (from `PENDING` only) |

Translations: `backend/src/resources/trans/{ar,en}/general.ts` → `enums.myFatoorahInvoiceStatus`.

### 3.3 `related_data` union

```ts
type MyFatoorahInvoiceRelatedDataSubscription = {
  pay_for: "SUBSCRIPTION"
  plan_id: number
  plan_billing_period: "MONTHLY" | "YEARLY"
  subscription_id?: number  // written after successful finalize
}

type MyFatoorahInvoiceRelatedData =
  | MyFatoorahInvoiceRelatedDataSubscription
```

- Soft ids only inside arms — **no** ORM FK to `Plan` or `Subscription`.
- Optional `subscription_id` is the soft link after entitlement create (idempotency marker).

### 3.4 Indexes

- unique `private_key`
- unique `mf_invoice_id`
- `customer_id`
- `(customer_id, status)`

### 3.5 Relations

| Side | Association |
|---|---|
| Invoice | `belongsTo(Customer)` |
| Customer | `hasMany(MyFatoorahInvoice)` |

## 4) Write path: `subscription.subscribe`

File: `backend/src/app/orchestrator/requesters/SubscriptionRequester.ts`

- `@requester("subscription")` / `@sub(["website"], ["customer"])`
- Method name: **`subscribe`**
- Map: `backend/requesters.website.ts` → `customer.subscription: "subscribe"`
- Website mirror: `website/src/types/requesters/requesters.website.ts`

### 4.1 Input

| Field | Validation |
|---|---|
| `plan` | `isPlan(joi)` → `Plan().Opt` (model ref, not bare `plan_id`) |
| `plan_billing_period` | `MONTHLY` \| `YEARLY` |
| `method_id` | positive integer |

Facts: `isByCustomerFacts(joi)`.

### 4.2 Execution (one transaction)

1. `customer.can("SUBSCRIPTION", { sub: "subscribe", plan, throwMode: true })`
2. `resolveSubscriptionPlanAmount(plan, period)` — invalid/≤0 → `"NOT_VALID"`
3. Create invoice `PENDING` + `related_data` subscription arm + temporary `uuid()` `private_key`
4. `MyFatoorah.createPaymentUrl({ …, platform: "website" })`
5. Persist `private_key` (from helper), `mf_invoice_id`, `mf_method_id`, `mf_payment_url`
6. Return `{ paymentURL, invoiceId }` in form `other` + `SUCCESS_CREATE`

Still **no** `Subscription` row at this step.

## 5) Finalize path (entitlement)

File: `backend/src/app/http/controllers/external/payments/MyFatoorahFinalizeController.ts`

Route: `GET /external/payments/my_fatoorah/finalize/:platform(website)/:status(success|error)/:privateKey`

### 5.1 Transition rules

Remote `MyFatoorah.getPaymentStatus(mf_invoice_id)` drives outcome:

| Condition | Result |
|---|---|
| Remote `Paid` | `COMPLETED` |
| Invoice already `COMPLETED` | `COMPLETED` (idempotent) |
| Remote `Canceled` | `CANCELED` |
| Callback `error` (else) | `FAILED` |
| Default | `FAILED` |

Optional query `paymentId` / `PaymentId` / `payment_id` → `mf_payment_id` when new.

### 5.2 First `COMPLETED`

When status was not already `COMPLETED` and `related_data.pay_for === "SUBSCRIPTION"` and `subscription_id` is absent:

1. Load Plan by `related_data.plan_id` (missing → `"UNEXPECTED_ERROR"`)
2. `Subscription.subscribe({ customerId, plan, planPrice: Number(amount), planBillingPeriod, transaction })`
3. Patch `related_data.subscription_id`
4. Set invoice `status: COMPLETED`

`FAILED` / `CANCELED` only move from `PENDING`.

### 5.3 Website redirect

| Outcome | HTTP |
|---|---|
| Success path | 302 `{WEBSITE_URL}/customer?paymentReturn=success` |
| Error / exception on website platform | 302 `…?paymentReturn=error` |
| Missing `WEBSITE_URL` | 500 `"WEBSITE_URL"` |

Non-website platform (path currently only allows `website`): 200 JSON `{ finalizedTo, invoiceId }` or plain 404/500 string.

### 5.4 Failure modes

| Signal | When |
|---|---|
| `"404"` | Invoice not found by `private_key` |
| `"UNEXPECTED_ERROR"` | Missing `mf_invoice_id`, or Plan missing at fulfill |
| `"WEBSITE_URL"` | Env unset on website redirect |

## 6) Entitlement create: `Subscription.subscribe`

File: `backend/src/app/orm/models/Subscription.ts` — **static** `subscribe(op)`.

Behavior:

- `starts_at = now`
- `ends_at` = calendar `+planBillingPeriodCount` months (`MONTHLY`) or years (`YEARLY`) via `moment`
- Prior customer rows with `status: ACTIVE` → `REPLACED`
- Create new `ACTIVE` with Plan limit snapshot + `plan_price` from caller (finalize passes invoice `amount`)

Callers must use this static; do not duplicate replace+create inline. Detail: `subscription-domain.md` §3.8.

## 7) Ability gate

Owner: `Customer.Ability.SUBSCRIPTION`

```ts
SUBSCRIPTION: {
  sub: "subscribe"
  plan?: ModelRef<PlanModel>
}
```

`can("SUBSCRIPTION", { sub: "subscribe", plan })`:

- resolve plan (missing ref → general allow / no target checks; provided but not found → `"404"`)
- plan `status !== "ACTIVE"` → `"ACTION_NOT_ALLOWED"`

GQL exposure: `_Me.canSubscribe(planId: ID!): _Ability` via `MeBridge` extras + `visualMode`.

Requester still enforces with `throwMode: true` even when GQL returns `CAN`.

## 8) GraphQL (customer reads only)

### 8.1 `_PaymentMethod` (`base.graphql`)

Fields: `id!`, `name`, `image_url`, `service_charge`, `total_amount`, `currency_iso`, `payment_method_code`, `is_direct_payment`.

### 8.2 Root `subscriptionPaymentMethods`

SDL: `subscriptionPaymentMethods(planId: ID!, billingPeriod: _PlanBillingPeriodValue!): [_PaymentMethod]`

Helper: `backend/src/app/helpers/CustomerSubscriptionPaymentMethods.ts`

- ACTIVE Plan required; else `[]`
- Amount from `monthly_price` / `yearly_price`; invalid → `[]`
- `MyFatoorah.memoGetPaymentsMethods({ amount, lang })`
- Thin resolver in `CustomerSchema` — **no Bridge**

### 8.3 No invoice GQL

Do not add invoice list/detail roots or bridges unless product explicitly requests them.

## 9) HTTP mount `/external`

See `external-http-mount-and-myfatoorah-callbacks.md`.

| Item | Value |
|---|---|
| Router | `backend/src/app/http/routes/external.ts` |
| Base controller | `ExternalControllerBase` |
| Middleware group | `external` = `compression` + `json` only |
| Controllers registry | local `backend/.types/controllers.ts` → `external.payments.MyFatoorahFinalizeController` |

Env used by payment stack:

| Env | Role |
|---|---|
| `PAYMENT_MODE` | `test` / `live` MF host selection |
| `MYFATORAH_TEST` / `MYFATORAH_LIVE` | Bearer tokens |
| `EXPRESS_BASE_URL` | Absolute callback URLs via `route("/external", …)` |
| `WEBSITE_URL` | Post-finalize 302 base |

## 10) Website mirrors

| Path | Role |
|---|---|
| `website/src/types/gql/definitions/base.graphql` | includes `_PaymentMethod` |
| `website/src/types/gql/definitions/customer.graphql` | `subscriptionPaymentMethods`, `canSubscribe` |
| `website/src/types/gql/gql-types/{base,customer}.ts` | generated mirrors |
| `website/src/types/requesters/requesters.website.ts` | `customer.subscription: "subscribe"` |

No website UI in this domain.

## 11) Verification

Existing scripts only:

- `backend/`: `yarn generate-types`, `yarn type-check`
- `website/`: `yarn type-check` after mirror copy

## 12) Traceability map

| Path | Status | Doc home |
|---|---|---|
| `backend/src/app/orm/models/MyFatoorahInvoice.ts` | added | §3 |
| `backend/src/app/orm/models/Customer.ts` | modified | §3.5, §7 |
| `backend/src/app/orm/models/Subscription.ts` | modified | §6; subscription-domain |
| `backend/src/app/orchestrator/requesters/SubscriptionRequester.ts` | added | §4 |
| `backend/requesters.website.ts` | modified | §4 |
| `backend/src/app/validation/joi_rules.ts` | modified | §4.1 `isPlan` |
| `backend/src/app/helpers/CustomerSubscriptionPaymentMethods.ts` | added | §8.2 |
| `backend/src/app/helpers/MyFatoorah.ts` | pre-existing | §9; external-http doc |
| `backend/src/app/http/routes/external.ts` | added | §9 |
| `backend/src/app/http/controllers/external/ExternalControllerBase.ts` | added | §9 |
| `backend/src/app/http/controllers/external/payments/MyFatoorahFinalizeController.ts` | added | §5 |
| `backend/src/resources/configs/http/express.ts` | modified | §9 |
| `backend/src/app/gql/definitions/base.graphql` | modified | §8.1 |
| `backend/src/app/gql/definitions/customer.graphql` | modified | §7, §8.2 |
| `backend/src/app/gql/gql-types/{base,customer,supervisor}.ts` | generated | §8 / §11 |
| `backend/src/app/gql/schemas/CustomerSchema.ts` | modified | §8.2 |
| `backend/src/app/gql/bridges/customer/MeBridge.ts` | modified | §7 |
| `backend/src/resources/trans/{ar,en}/general.ts` | modified | §3.2 |
| `backend/.types/models.ts` | local/gitignored | §3 — `"MyFatoorahInvoice"` |
| `backend/.types/controllers.ts` | local/gitignored | §9 — `external` tree |
| `website/src/types/gql/definitions/{base,customer}.graphql` | mirror | §10 |
| `website/src/types/gql/gql-types/{base,customer}.ts` | mirror | §10 |
| `website/src/types/requesters/requesters.website.ts` | mirror | §10 |
| `website/lib/tsconfig.tsbuildinfo` | build artifact | excluded from narrative |
| `.cursor/rules/myfatoorah-invoice-payment-domain.mdc` | added | durable invariants |
| `.cursor/rules/subscription-domain.mdc` | modified | `Subscription.subscribe` |
| `.cursor/rules/backend-external-http-mount.mdc` | pre-existing | §9 |
| `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md` | this file | all |

## 13) Related

- `docs/platforms/backend/contracts/subscription-domain.md`
- `docs/platforms/backend/contracts/plan-domain.md`
- `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `docs/platforms/backend/contracts/http-and-requesters.md`
- `docs/platforms/website/graphql-mirror-and-tooling.md`
- `.cursor/rules/myfatoorah-invoice-payment-domain.mdc`
- `.cursor/rules/subscription-domain.mdc`
- `.cursor/rules/backend-external-http-mount.mdc`
- `.cursor/rules/backend-able-can-gql-contract.mdc`
