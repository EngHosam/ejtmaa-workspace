# Client Portal HTTP — Website (Ejtmaa)

## Scope

Ejtmaa customer client portal: the SSR frontend on backend mount **`/website`**.

## Mount

- Path: `/website`
- Route module: `backend/src/app/http/routes/website.ts`
- Middleware: `website` group (granting resolves visitor or customer)

## Consumer

- Repository: `website/`
- Axios base: configured in `website/src/resources/configs/axios/api.ts`
- Forms: `FORMS.R` (visitor), `FORMS.CUSTOMER.R`
- GQL adapter: `DATA_ADAPTERS.GQL` (visitor), `DATA_ADAPTERS.CUSTOMER.GQL` (customer)
- Customer GQL controller: `backend/src/app/http/controllers/website/data_adapters/customer/GQLAdapterController.ts`
- Custom start: `CUSTOM.START` → `/website/custom/start`

## Start payload

`/website/custom/start` returns boot hydration including auth state under the `auth` key (SSR cookie-token model).

## Actor model

| Actor | Access |
|---|---|
| visitor | Public routes, auth forms |
| customer | Authed customer routes and GQL |

## Cpanel counterpart

Supervisor portal: `/cpanel` consumed by `cpanel/` repository.
See `docs/platforms/cpanel/overview.md`.
