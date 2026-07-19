# Website Platform Overview (Ejtmaa)

## Role

`website/` is the **Customer** SSR frontend for Ejtmaa, served on backend mount `/website`.

Actors: **visitor** (unauthenticated) and **customer** (authenticated).

## Architecture

Same `@my-ssr/web-core` family as `cpanel/`:
- `@my-ssr/web-core` + `@typescript/sys-core`
- `src/client/*`, `src/server/*`, `src/resources/configs/web-core.ts`
- Folder ownership: `resources/`, `services/`, `ui/base/`, `ui/components/`, `ui/layouts/`, `ui/pages/`, `types/`

## UI foundation

- `ui/base/components/Utils.tsx` — layout/styling primitives
- `resources/configs/theme.ts` — brand source of truth (navy `#0B2057`, orange `#EC6901`)
- `resources/configs/utils.ts` — theme path helpers

See `docs/platforms/website/ui-foundation.md`.

## Backend coupling

- Requesters: `visitor.auth`, `customer.customer`, `customer.notification`, `customer.subscription` (`subscribe`), `customer.member`, `customer.organization`, `customer.meeting` (`create`) (see `src/types/requesters/requesters.website.ts`)
- GQL mirrors: `base` + `customer` (`me` + nested `_Me.organization` / `_Me.currentSubscription` / `_Subscription`, `me.canDeleteNotifications`, `me.canSubscribe(planId)`, `notifications`, `members(filter: _MemberFilter)`, `member`, `messageChannels`, `messageChannel`, `messageTemplates`, `messageTemplate`, `meetings`, `meeting` with nested `participants` / `_MeetingParticipant`, `agendaItems` / `_AgendaItem`, `decisions` / `_Decision` / `votes` / `_Vote`, `talkRecords` / `_TalkRecord`, `plans` / `plan` / `_Plan`, `subscriptions` / `subscription`, `subscriptionPaymentMethods` / `_PaymentMethod`; no customer root `organization`)
- Socket namespace: `customer`
- Events: `OnCustomerEvent` (payload `OnCustomerEventDate { type: "UPDATED" }`)
- Backend mount: `/website` (test `http://192.168.1.10:3206/website`, prod `https://backend.ejtmaa.live/website`)

## Flow documentation

| Flow | Document |
|---|---|
| Auth | [flow-auth.md](./flow-auth.md) |
| Form foundation | [flow-form-foundation.md](./flow-form-foundation.md) |
| Settings | [flow-settings.md](./flow-settings.md) |
| Notifications | [flow-notifications.md](./flow-notifications.md) |
| Static info pages | [flow-static-info-pages.md](./flow-static-info-pages.md) |
| Customer shell | [flow-customer-shell.md](./flow-customer-shell.md) |
| Customer members (directory + form) | [flow-customer-members.md](./flow-customer-members.md) |
| Customer meetings (directory + create + empty details) | [flow-customer-meetings.md](./flow-customer-meetings.md) |
| Customer organization (settings form) | [flow-customer-organization.md](./flow-customer-organization.md) |

## 6) Route registry summary

### 6.1) Shipped routes

Authoritative table: `route-registry-contract.md` §1.1 / §5.2 (includes `CustomerHome`, `CustomerMembers`, multi-path `CustomerMemberForm`, `CustomerOrganization`).

`publicRoutes` = `["Login", "Register", "ResetPassword", "UiMockup", "Home"]`. `getMyHomeIdentify` returns `"CustomerHome"` when `authedAs === "CUSTOMER"`, else `"Home"`.

Layouts shipped: `BasicLayout` (`BASIC`), `LandingLayout` (`LANDING`), `MainLayout` (`MAIN`), `CustomerMainLayout` (`CUSTOMER_MAIN`).

Shipped customer workspace: `CustomerHome` (`/customer`), `CustomerMeetings` (`/customer/meetings`), `CustomerMeetingForm` (`/customer/meetings/form`), `CustomerMeetingDetails` (`/customer/meetings/:id`), `CustomerMembers` (`/customer/members`), `CustomerMemberForm` (`/customer/members/form` + `/:id`), `CustomerOrganization` (`/customer/organization`), `CustomerMessageChannels` (`/customer/message-channels`), `CustomerMessageChannelForm` (`/customer/message-channels/form` + `/:id`). All `mustAuthedAs: ["CUSTOMER"]`, `CUSTOMER_MAIN`. Shell: `flow-customer-shell.md`. Meetings: `flow-customer-meetings.md`. Members: `flow-customer-members.md`. Organization: `flow-customer-organization.md`. Message channels: `flow-customer-message-channels.md`. Forms: `flow-form-foundation.md`.

### 6.2) Planned (not shipped)

Remaining `/customer/*` workspace routes (message templates, subscription, settings, support, help, notifications, static info, bottom bar) — see `route-registry-contract.md` §5.2 and `flow-customer-shell.md`. Meetings, members, organization, and message channels are **shipped** (see §6.1).

## Documentation stance

Platform docs under `docs/platforms/website/` describe the customer portal contract: routes, GQL mirrors, requesters, flows, and UI ownership for visitor + customer actors on `/website`.

## Related

- `docs/platforms/website/README.md` — index
- `docs/platforms/website/repository-inventory.md` — file inventory
- `docs/invariants/website.md` — invariants
- `.cursor/rules/website-platform-governance.mdc` — governance rule
