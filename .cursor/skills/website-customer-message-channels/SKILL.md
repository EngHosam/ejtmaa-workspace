---
name: website-customer-message-channels
description: >-
  Ships or extends the website customer message-channels directory
  (CustomerMessageChannels, ResultLane, FiSend drawer tile before Settings).
  Use when changing channel list UI, drawer placement, or channel card chrome.
---

# Website customer message channels directory

## When to Use

- Adding or changing `/customer/message-channels` list UI.
- Reordering the message-channels drawer tile relative to Settings.
- Changing `CustomerMessageChannelCard` chrome or GQL selection for the list.

## Instructions

1. Read `docs/platforms/backend/contracts/message-channel-domain.md` and `docs/platforms/website/flow-customer-message-channels.md`.
2. Keep route `CustomerMessageChannels` on `CUSTOMER_MAIN` with breadcrumb parent `CustomerHome` (W38 / breadcrumb-subpage skill).
3. Drawer: identify `CustomerMessageChannels`, glyph **`FiSend`**, order **after** Subscription / **before** Settings. Same `FiSend` on the card icon well — `.cursor/rules/website-customer-section-glyph-consistency.mdc`.
4. Adapter: mount-private `"customer-message-channels"` inheriting `DATA_ADAPTERS.CUSTOMER_GQL`, `listable: "messageChannels"`. No history search until `_MessageChannelFilter` exists.
5. Card is presentational (W42): screen passes `name`, `typeLabel`, `statusLabel`, optional `detailLabel`. Do not call `useTranslator` inside the card.
6. Do not select or display `smtp_password` / `adwhats_token`.
7. Section subtitle/empty copy may say **WhatsApp**; card type labels come from backend enum (`ADWHATS` → أد واتس / Ad Whats).
8. No Add/form chrome until a channel requester ships.
9. Update `flow-customer-message-channels.md`, `flow-customer-shell.md` §5.3, and `route-registry-contract.md` §5.2 in the same change.
10. Verify: website `yarn type-check`. Style props: Utils shorthands over `cssStyle` when available (`website-utils-style-prop-audit`).

## Related

- `.cursor/skills/website-customer-drawer-nav/SKILL.md`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/rules/website-presentational-label-props.mdc`
