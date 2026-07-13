# External HTTP mount: `/external` (target contract)

Payment gateway callback mount contract for Ejtmaa.

`MyFatoorah.ts` already builds callback URLs under `/external`. This document defines the mount, middleware group, routes, and website redirect contract to land when payment finalize controllers ship.

Activation status:
- `backend/src/resources/configs/http/express.ts` mounts `/website` and `/cpanel`.
- `/external` router registration and `routes/external.ts` complete this contract when payment finalize controllers ship.

## 1) Purpose

- **Dedicated callback surface**: MyFatoorah `CallBackUrl` and `ErrorUrl` target `/external/...` so redirects bypass `/website` and `/cpanel` middleware stacks.
- **Platform in the URL path**: Callers use `platform: "website"` in `MyFatoorah.createPaymentUrl`.
- **Finalize ownership**: the HTTP handler finalizes payment state, then redirects the website client or renders a done page.

## 2) Mount configuration (target)

**File**: `backend/src/resources/configs/http/express.ts`

| Path prefix | Router module | Middleware group |
|---|---|---|
| `/external` | `backend/src/app/http/routes/external.ts` | `external` |

`route("/external", suffix)` from `export const route = _route("express")` builds absolute callback URLs for `MyFatoorah.createPaymentUrl`.

## 3) Middleware group `external` (target)

Included: `compression`, `json`.

Excluded by design: `query_values`, `granting`, `local`, `errors_funnel`.

Unhandled controller errors on this mount reach the global `errorsFunnel` handler with `req.errorsFunnel` undefined, producing `{ error: 0 }` per `express.ts` configuration.

## 4) HTTP surface (target)

| Method | Path (under `/external`) | Controller |
|---|---|---|
| `GET` | `/payments/my_fatoorah/finalize/:platform(website)/:status(success\|error)/:privateKey` | `c.external.payments.MyFatoorahFinalizeController` |
| `GET` | `/*` | Inline JSON 404 |

## 5) MyFatoorah helper (live)

**File**: `backend/src/app/helpers/MyFatoorah.ts`

`createPaymentUrl(op)` requires `platform: "website"` on `MakePaymentURLOptions`.

Callback URLs sent to MyFatoorah ExecutePayment:

- `CallBackUrl`: `route("/external", `/payments/my_fatoorah/finalize/${platform}/success/${privateKey}`)`
- `ErrorUrl`: `route("/external", `/payments/my_fatoorah/finalize/${platform}/error/${privateKey}`)`

Returned shape: `{ InvoiceId, PaymentURL, privateKey }`.

## 6) Website redirect (target)

After finalize transitions for `platform: "website"`:

| Behavior | Detail |
|---|---|
| Success path | HTTP **302** to `{WEBSITE_URL}/customer?paymentReturn=success` |
| Error path | HTTP **302** with `paymentReturn=error` |
| Missing env | If `process.env.WEBSITE_URL` is missing/empty, controller responds `500 "WEBSITE_URL"` |

Post-return client handling is owned by the website route that initiated payment.

## 7) Related

- `.cursor/rules/backend-external-http-mount.mdc`
- `docs/platforms/backend/contracts/http-and-requesters.md`
