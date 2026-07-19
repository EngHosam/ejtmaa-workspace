---
name: website-customer-message-channels
description: >-
  Ships or extends the website customer message-channels directory and form
  (CustomerMessageChannels, CustomerMessageChannelForm, ResultLane, FiSend
  drawer tile before Settings). Use when changing channel list/form UI,
  drawer placement, or channel card chrome.
---

# Website customer message channels

## When to Use

- Adding or changing `/customer/message-channels` list or form UI.
- Reordering the message-channels drawer tile relative to Settings.
- Changing `CustomerMessageChannelCard` chrome or GQL selection for the list.

## Instructions

1. Read `docs/platforms/backend/contracts/message-channel-domain.md` and `docs/platforms/website/flow-customer-message-channels.md`.
2. Confirm backend `MessageChannelRequester` + `Customer.Ability.MESSAGE_CHANNEL` + `requesters.website` map (`read` | `create` | `update` | `delete`).
3. List route `CustomerMessageChannels` on `CUSTOMER_MAIN` with breadcrumb parent `CustomerHome`. Form: multi-path `CustomerMessageChannelForm` after the list identify — `.cursor/rules/website-multi-path-form-routes.mdc`.
4. Drawer: identify `CustomerMessageChannels`, glyph **`FiSend`**, order **after** Subscription / **before** Settings. Same `FiSend` on the card — `.cursor/rules/website-customer-section-glyph-consistency.mdc`.
5. Register `Forms.CUSTOMER_MESSAGE_CHANNEL` → `API.FORMS.CUSTOMER.R("messageChannel")`. Href via `buildCustomerMessageChannelFormHref`.
6. Adapter: mount-private `"customer-message-channels"` inheriting `DATA_ADAPTERS.CUSTOMER_GQL`. No history search until `_MessageChannelFilter` exists.
7. Card is presentational (W42): screen passes labels + optional `editLabel`/`onEdit`.
8. Form: `formType = id ? "update" : "create"`; type chooser on create only; update shows locked type label; credentials by `type`; status not on UI. Do not call `testConnection` from UI.
9. Delete: `await confirm(…, "danger")` — `.cursor/skills/website-confirm-modal/SKILL.md`. Loading: `saving` / `deleting` from `currentSub` (`flow-form-foundation.md` §3.10).
10. Create: stable `formIdentify` + `d.reset()` on create success before `nav.back()`. Do not echo `messageChannel` id from requester `read`.
11. Section subtitle/empty copy may say **WhatsApp**; card type labels come from backend enum.
12. Update `flow-customer-message-channels.md`, `flow-form-foundation.md`, and `route-registry-contract.md` when contracts change.
13. Verify: website `yarn type-check` (and backend if requester/Ability/GQL changed).

## Related

- `.cursor/skills/website-customer-member-form/SKILL.md`
- `.cursor/skills/website-confirm-modal/SKILL.md`
- `.cursor/skills/website-customer-drawer-nav/SKILL.md`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/rules/website-presentational-label-props.mdc`
- `.cursor/rules/website-confirm-modal.mdc`
- `.cursor/rules/message-channel-domain.mdc`
