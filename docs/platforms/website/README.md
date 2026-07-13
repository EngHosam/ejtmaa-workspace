# Website Documentation Index

## Foundation

| Document | Scope |
|---|---|
| [`overview.md`](overview.md) | Customer portal architecture |
| [`repository-inventory.md`](repository-inventory.md) | Repo file inventory |
| [`component-structure.md`](component-structure.md) | Folder and layer ownership |
| [`ui-foundation.md`](ui-foundation.md) | Utils + theme.ts contract |
| [`brand-identity-alignment.md`](brand-identity-alignment.md) | URL rebrand + semanticColor audit outcome |
| [`data-flow-and-gql.md`](data-flow-and-gql.md) | Adapters, requesters, GQL |
| [`graphql-mirror-and-tooling.md`](graphql-mirror-and-tooling.md) | GQL mirror sync |
| [`route-registry-contract.md`](route-registry-contract.md) | Customer route registry |
| [`ssr-boot-and-startup.md`](ssr-boot-and-startup.md) | SSR boot lifecycle |
| [`shared-ui-and-shell.md`](shared-ui-and-shell.md) | Shell components |
| [`page-error.md`](page-error.md) | Error (403/404/500) page composition |

## Flows

| Flow | Document | Scope |
|---|---|---|
| Auth | [`flow-auth.md`](flow-auth.md) | Customer auth on `/website` |
| Form foundation | [`flow-form-foundation.md`](flow-form-foundation.md) | Shared form patterns |
| Settings | [`flow-settings.md`](flow-settings.md) | Account settings |
| Notifications | [`flow-notifications.md`](flow-notifications.md) | Notification list |
| Static info | [`flow-static-info-pages.md`](flow-static-info-pages.md) | About Ejtmaa, legal pages |
| Customer shell | [`flow-customer-shell.md`](flow-customer-shell.md) | Authed customer layout |

## Invariants

- [`../../invariants/website.md`](../../invariants/website.md)

## Change set traceability

Current-state reflection of the brand-polish change set (radius tokens, logo sizing/no-frame, Error page redesign). No change history.

| Path (under `website/`) | Documented where |
|---|---|
| `src/resources/configs/theme.ts` (`Dims` corner radius) | `ui-foundation.md` § Corner radius tokens; `brand-identity-alignment.md`; invariant W46; `website-corner-radius-tokens.mdc` |
| `src/resources/configs/utils.ts` (`semanticDims.card.radius`) | `ui-foundation.md` § Corner radius tokens; `page-error.md` § Radius + tokens; `website-corner-radius-tokens.mdc` |
| `src/app/ui/components/Logo.tsx` (presets + sizes) | `brand-identity-alignment.md` § Logo; `shared-ui-and-shell.md` § 3/4; invariant W45; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Header.tsx` (logo no-frame) | `shared-ui-and-shell.md` § 3; `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Footer.tsx` (logo no-frame) | `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/components/Drawer.tsx` (logo no-frame) | `shared-ui-and-shell.md` § 4; `brand-identity-alignment.md` § Logo; `website-logo-no-frame.mdc` |
| `src/app/ui/pages/Error.tsx` (full page) | `page-error.md` (full composition + brand treatment); `brand-identity-alignment.md` § Canonical consumer pairings |
| `src/resources/translations/ar.ts` (`error.title` key) | `page-error.md` § Translation contract; `page-error.md` § Page title |
| `public/images/{dark,light}_logo.png` (binary brand assets) | `brand-identity-alignment.md` § Logo (asset paths + scheme swap). Binary; not narrated line-by-line. |
| `lib/tsconfig.tsbuildinfo` | Generated build artifact from `yarn type-check`; not narrated. |

`ar.ts` also carries a brand-rename pass (`مصدرية` → `اجتماع` in `app.title`, `footerTitle`, and `brand` strings), made outside this agent session; reflected as the current app title value, not narrated key-by-key.

## Governance

- `.cursor/rules/website-platform-governance.mdc`
- `.cursor/skills/website-platform-governance/SKILL.md`
