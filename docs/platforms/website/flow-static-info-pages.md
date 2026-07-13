# Website Static Info Pages

Customer portal contract (see `overview.md`).

## Routes

| Identify | Path | Layout | Breadcrumb parent |
|---|---|---|---|
| `CustomerHelpGuide` | `/customer/help-guide` | `CUSTOMER_MAIN` | `CustomerHome` |
| `CustomerAbout` | `/customer/about` | `CUSTOMER_MAIN` | `CustomerHome` |
| `CustomerTerms` | `/customer/terms` | `CUSTOMER_MAIN` | `CustomerHome` |

Entry: customer drawer `supportAndInfo` section. Customer-only routes under `/customer/*`.

## Architecture

| Layer | Path |
|---|---|
| Thin `MyPage` | `ui/pages/customer/{HelpGuide,About,Terms}.tsx` |
| Help wiring | `ui/components/customer/static/CustomerHelpGuideScreen.tsx` |
| Shared screens | `ui/components/static/{HelpGuideScreen,AboutEjtmaaScreen,TermsConditionsScreen}.tsx` |

No backend, GQL, adapters, or forms — i18n only.

## Translation namespaces

| Surface | Namespace |
|---|---|
| Customer help | `ui.pages.customer.helpGuide` |
| About Ejtmaa | `ui.pages.shared.aboutEjtmaa` |
| Terms | `ui.pages.shared.termsConditions` |
| Drawer labels | `ui.layouts.customerMainLayout.drawer` |

## UI contract

- `Container` + `pt={"2rem"}` + `SectionHeading` alongside breadcrumb sub-header
- `Helmet` title from scoped translator
- Help guide quick-link tiles are informational only — no `nav.push`
- Directional icons use `Icn` `flp` per `.cursor/rules/website-icon-rtl-flip.mdc`

## Related

- `.cursor/rules/website-static-info-pages.mdc`
- `docs/platforms/website/flow-customer-shell.md`
