# Website Flow — Customer Message Channels (Directory)

Authenticated customer org delivery-channel directory on `CUSTOMER_MAIN`. Shell/breadcrumb contract: `flow-customer-shell.md` §7.1. Backend read contract: `docs/platforms/backend/contracts/message-channel-domain.md`.

## 1) Scope

**Shipped**

- Message-channel directory for the authenticated customer's organization (GQL list + ResultLane load-more).
- Route `CustomerMessageChannels` → `/customer/message-channels` with breadcrumb parent `CustomerHome`.
- Drawer tile `itemMessageChannels` / `CustomerMessageChannels` / **`FiSend`**, ordered **after** Subscription and **immediately before** Settings.
- Presentational card chrome aligned with meeting cards (accent rail, `cardBackground`, `shellBorder`, `semanticDims.card.radius`).
- Card glyph **`FiSend`** — same Feather icon as the drawer tile (section identity consistency).
- Meta line: `typeLabel · statusLabel` from GQL enum labels (`messageChannelType` / `messageChannelStatus`), e.g. أد واتس برو · نشط.
- Optional detail line: `from_address` (email) or `adwhats_account_id` (WhatsApp channels).
- Labels for type/status/detail resolved on the **screen** and passed as props (W42) — card has no `useTranslator`.
- Secrets never selected in the list query (`smtp_password`, `adwhats_token`).
- User-facing **section** copy (subtitle / empty) says **WhatsApp** (product language), not “Ad Whats”. Backend enum labels for `ADWHATS` / `ADWHATS_PRO` remain أد واتس / أد واتس برو on the card meta line.

**Not shipped**

- Create / edit / delete form or `MessageChannelRequester`.
- History search / type-status filter chips (no `_MessageChannelFilter` yet).
- Channel details route / card link.
- Connectivity test UX / status flip on send failure.
- Temporary design-preview cards (removed after visual review).

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| List route | `CustomerMessageChannels` → `/customer/message-channels` |
| List page | `website/src/app/ui/pages/customer/CustomerMessageChannels.tsx` |
| List screen | `website/src/app/ui/components/customer/message-channels/CustomerMessageChannelsScreen.tsx` |
| Hook | `website/src/app/ui/components/customer/hooks/useCustomerMessageChannels.ts` |
| Card | `…/message-channels/CustomerMessageChannelCard.tsx` |
| Skeleton | `…/message-channels/MessageChannelCardSkeleton.tsx` |
| Drawer | `website/src/app/ui/components/customer/CustomerDrawer.tsx` |

Layout: `CUSTOMER_MAIN`, `mustAuthedAs: ["CUSTOMER"]`. Breadcrumb: `{ parent: "CustomerHome", label: tr => tr.ui.pages.customer.messageChannels.title }`.

## 3) Directory data adapter

| Concern | Value |
|---|---|
| Mount-private adapter id | `"customer-message-channels"` |
| Inherited | `DATA_ADAPTERS.CUSTOMER_GQL` → `API.DATA_ADAPTERS.CUSTOMER.GQL` |
| Listable | `"messageChannels"` |
| Default page size | `initDataAdaptersProps.default.maxLoadLength` = **24** |
| Reload | `mLoad({ reload: true, query })` on mount when adapter `exist`; same `query` on load-more / refresh |
| History search | **none** (no GQL filter yet) |

GQL (inline in hook):

```graphql
query CustomerMessageChannels {
    messageChannels {
        id
        name
        type {
            value
            label
        }
        status {
            value
            label
        }
        smtp_host
        from_address
        adwhats_account_id
        total_count
    }
}
```

Note: `smtp_host` is selected but not currently rendered on the card (detail prefers `from_address` / `adwhats_account_id`).

## 4) Screen composition

`Container` → `Col pt={2} gap={1.5} pb={2}`:

1. `Helmet` title + `SectionHeading` (title + subtitle) — **no** Add button (write path not shipped).
2. `ResultLane` → `CustomerMessageChannelCard` (not a `Link`; no details route).

Card props from screen:

| Prop | Source |
|---|---|
| `name` | `channel.name` |
| `typeLabel` | `channel.type.label` \|\| `value` |
| `statusLabel` | `channel.status.label` \|\| `value` |
| `detailLabel` | `from_address` \|\| `adwhats_account_id` (trimmed; omitted if empty) |

Utils style: accent rail + icon well use shorthands (`overlay`, `flx={"0 0 auto"}`); skeleton `animation` stays in `cssStyle` (no shorthand). No `baseCssStyle` on the card.

## 5) Drawer IA

Order excerpt (`flow-customer-shell.md` §5.3):

| Order | Identify | Glyph |
|---|---|---|
| … | `CustomerSubscription` | `FiCreditCard` |
| **7** | `CustomerMessageChannels` | **`FiSend`** |
| 8 | `CustomerSettings` | `FiSettings` |

Route gating: tile enabled only when `routes` has `CustomerMessageChannels`.

## 6) i18n

| Key area | Purpose |
|---|---|
| `ui.layouts.customerMainLayout.drawer.itemMessageChannels` | Drawer label (قنوات الإرسال / Message channels) |
| `ui.pages.customer.messageChannels.title` | Page + breadcrumb + Helmet |
| `…subtitle` / `emptyTitle` / `emptyDescription` | Product copy uses **WhatsApp** wording |
| `…loadMoreBtn` / `loadingMoreHint` | ResultLane chrome |

ar/en must stay mirrored.

## 7) Failure modes

| Condition | Behavior |
|---|---|
| GQL load error | ResultLane wrong overlay + retry → `refresh` |
| Empty list | empty title/description |
| Unauthed | `mustAuthedAs: ["CUSTOMER"]` |

## 8) Traceability map

| Path | Role | Section |
|---|---|---|
| `website/src/resources/configs/routes.ts` | Route + `MPagesRoutes` | §2 |
| `website/src/app/ui/components/customer/CustomerDrawer.tsx` | Drawer tile + `FiSend` | §5 |
| `website/src/app/ui/pages/customer/CustomerMessageChannels.tsx` | Thin `MyPage` | §2 |
| `website/src/app/ui/components/customer/message-channels/CustomerMessageChannelsScreen.tsx` | Screen + W42 labels | §4 |
| `website/src/app/ui/components/customer/message-channels/CustomerMessageChannelCard.tsx` | Presentational card | §4 |
| `website/src/app/ui/components/customer/message-channels/MessageChannelCardSkeleton.tsx` | Skeleton shape | §4 |
| `website/src/app/ui/components/customer/hooks/useCustomerMessageChannels.ts` | Adapter hook | §3 |
| `website/src/resources/translations/ar.ts` / `en.ts` | Drawer + page copy | §6 |
| `website/lib/tsconfig.tsbuildinfo` | Build artifact | excluded from narrative — local TS build cache |
| `docs/platforms/website/flow-customer-shell.md` | Drawer order table | §5 |
| `docs/platforms/website/route-registry-contract.md` | Registry §5.2 | indexes |
| `docs/platforms/website/overview.md` / `component-structure.md` / `data-flow-and-gql.md` / `README.md` | Indexes | indexes |
| `docs/platforms/backend/contracts/message-channel-domain.md` | Backend ORM/GQL | related |
| `.cursor/rules/website-customer-section-glyph-consistency.mdc` | Drawer ↔ card icon | governance |
| `.cursor/rules/website-presentational-label-props.mdc` | W42 channel card | governance |
| `.cursor/skills/website-customer-message-channels/SKILL.md` | Repeatable directory slice | governance |

## Related

- `docs/platforms/backend/contracts/message-channel-domain.md`
- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/website/route-registry-contract.md`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
- `.cursor/skills/website-customer-drawer-nav/SKILL.md`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/skills/website-utils-style-prop-audit/SKILL.md`
