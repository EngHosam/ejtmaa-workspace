# External HTTP mount: `/external`

Payment gateway callback mount contract for Ejtmaa.

`MyFatoorah.ts` builds callback URLs under `/external`. This document covers the live mount, middleware group, finalize route, and website redirect. Invoice + entitlement behavior detail: `myfatoorah-invoice-payment-domain.md`.

## 1) Purpose

- **Dedicated callback surface**: MyFatoorah `CallBackUrl` and `ErrorUrl` target `/external/...` so redirects bypass `/website` and `/cpanel` middleware stacks.
- **Platform in the URL path**: Callers use `platform: "website"` in `MyFatoorah.createPaymentUrl`.
- **Finalize ownership**: HTTP handler loads invoice by `private_key`, confirms remote MF status, transitions invoice `PENDING` → `COMPLETED` / `FAILED` / `CANCELED`, and on first `COMPLETED` for a subscription arm creates entitlement via `Subscription.subscribe`.

## 2) Mount configuration

**File**: `backend/src/resources/configs/http/express.ts`

| Path prefix | Router module | Middleware group |
|---|---|---|
| `/external` | `backend/src/app/http/routes/external.ts` | `external` |

`route("/external", suffix)` from `export const route = _route("express")` builds absolute callback URLs for `MyFatoorah.createPaymentUrl`.

## 3) Middleware group `external`

Included: `compression`, `json`.

Excluded by design: `query_values`, `granting`, `local`, `errors_funnel`.

Unhandled controller errors on this mount reach the global `errorsFunnel` handler with `req.errorsFunnel` undefined, producing `{ error: 0 }` per `express.ts` configuration.

## 4) HTTP surface

| Method | Path (under `/external`) | Controller |
|---|---|---|
| `GET` | `/payments/my_fatoorah/finalize/:platform(website)/:status(success\|error)/:privateKey` | `c.external.payments.MyFatoorahFinalizeController` |
| `GET` | `/*` | Inline JSON 404 |

Controller files:

- `backend/src/app/http/controllers/external/ExternalControllerBase.ts`
- `backend/src/app/http/controllers/external/payments/MyFatoorahFinalizeController.ts`

Local registry (gitignored): `backend/.types/controllers.ts` must expose the `external` tree after serve/build.

## 5) MyFatoorah helper

**File**: `backend/src/app/helpers/MyFatoorah.ts`

`createPaymentUrl(op)` requires `platform: "website"` on `MakePaymentURLOptions`.

Callback URLs sent to MyFatoorah ExecutePayment:

- `CallBackUrl`: `route("/external", `/payments/my_fatoorah/finalize/${platform}/success/${privateKey}`)`
- `ErrorUrl`: `route("/external", `/payments/my_fatoorah/finalize/${platform}/error/${privateKey}`)`

Returned shape: `{ InvoiceId, PaymentURL, privateKey }`.

Env: `PAYMENT_MODE`, `MYFATORAH_TEST` / `MYFATORAH_LIVE`, `EXPRESS_BASE_URL` (for absolute `route` URLs).

## 6) Finalize behavior (summary)

1. Find invoice by `private_key` (missing → `"404"`).
2. Require `mf_invoice_id`; call `MyFatoorah.getPaymentStatus`.
3. Resolve transition (remote `Paid` → `COMPLETED`; remote `Canceled` → `CANCELED`; callback `error` → `FAILED`).
4. Optionally persist `mf_payment_id` from query.
5. On first `COMPLETED` + subscription `related_data` without `subscription_id`: `Subscription.subscribe` + write soft `subscription_id`.
6. Idempotent when invoice status already equals target transition.

Full rules: `myfatoorah-invoice-payment-domain.md` §5–§6.

## 7) Website redirect

After finalize for `platform: "website"`:

| Behavior | Detail |
|---|---|
| Success path | HTTP **302** to `{WEBSITE_URL}/customer?paymentReturn=success` |
| Error path | HTTP **302** with `paymentReturn=error` |
| Missing env | If `process.env.WEBSITE_URL` is missing/empty, controller responds `500 "WEBSITE_URL"` |

Post-return client handling is owned by the website (not shipped in the payment backend package).

## 8) Related

- `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md`
- `.cursor/rules/backend-external-http-mount.mdc`
- `.cursor/rules/myfatoorah-invoice-payment-domain.mdc`
- `docs/platforms/backend/contracts/http-and-requesters.md`
