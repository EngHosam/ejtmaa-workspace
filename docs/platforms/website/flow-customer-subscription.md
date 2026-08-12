# Website Flow — Customer Subscription

Authenticated customer **Plan** catalog + **Subscription** checkout on `CUSTOMER_MAIN`.

| Related | Path |
|---|---|
| Shell / breadcrumb pattern | `flow-customer-shell.md` (§2 drawer; §7.1 subheader/breadcrumb) |
| Form foundation | `flow-form-foundation.md` |
| Backend Plan catalog | `docs/platforms/backend/contracts/plan-domain.md` |
| Backend Subscription / subscribe | `docs/platforms/backend/contracts/subscription-domain.md` |
| MyFatoorah finalize redirect | `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md` |
| Active-subscription gates (approve + live entry) | §11; `meeting-domain.md` §9.1b–§9.1c |

Product names: **الباقة** = `Plan`, **الاشتراك** = `Subscription`. Never name UI or code `Package` for this domain.

## 1) Scope

**Shipped**

- Route `CustomerSubscription` → `/customer/subscription` (drawer tile unlocks when this identify exists in `routes.ts`).
- Zones: current entitlement (`useMe().currentSubscription`), Plan catalog (`ResultLane` + `skeletonCount={3}`), checkout after **اشتراك** (period chips + payment methods + inline **دفع**).
- Cancel restores catalog (`FormActionButton` `tone="neutral"`).
- GQL reads: `plans`, `subscriptionPaymentMethods(planId, billingPeriod)`, `me.canSubscribe(planId)`.
- Write: `Forms.CUSTOMER_SUBSCRIPTION` on the screen (`useShallowForm`) → `subscribe` → `res.data.other.paymentURL` → same-tab `window.location.assign`.
- Post-pay return: query `paymentReturn=success|error` → manual toast + `useMe().update()` + `nav.push` replace without query (**this page only**).
- Money chrome: shared `SarMark` / `SarAmount` on plan cards, current card, and SAR method amounts.

**Not shipped**

- Invoice history / GQL invoice list.
- Soft-limit enforcement UI.
- Home handling of `paymentReturn` (single owner: this page).
- Home attention `no_subscription` → navigate to this route (item has no `href` today).
- Distinct Meeting linking copy for `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` (handshake still maps to generic linking `FAILED` / session `error = "NOT_VALID"`).
- Cpanel Plan admin.

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| Route | `CustomerSubscription` → `/customer/subscription` |
| Page | `website/src/app/ui/pages/customer/CustomerSubscription.tsx` |
| Screen | `…/subscription/CustomerSubscriptionScreen.tsx` |
| Presentational | `CustomerSubscriptionCurrentCard`, `CustomerSubscriptionPlanCard`, `CustomerSubscriptionMethodRow` + skeletons |
| Shared money | `website/src/app/ui/components/SarMark.tsx`, `SarAmount.tsx` |
| Hooks | `useCustomerSubscriptionPlans`, `useCustomerSubscriptionPaymentMethods`, `useCustomerSubscriptionCanSubscribe`, `useMe` |
| Form registry | `Forms.CUSTOMER_SUBSCRIPTION` → `API.FORMS.CUSTOMER.R("subscription")` |
| Toast | `website/src/app/services/toast.tsx` (`showToast`) |
| Drawer | `CustomerDrawer.tsx` (`CustomerSubscription` / `FiCreditCard` / `itemSubscription`) |

`layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`, breadcrumb parent `CustomerHome`.

`MPagesRoutes`:

```ts
CustomerSubscription: {
    query?: {
        paymentReturn?: string;
    };
};
```

## 3) Screen zones

| Zone | Behavior |
|---|---|
| Current | `me.currentSubscription` → card (plan name, period chip, `SarAmount` on `plan_price`, status, relative end via `useMoment` day-diff labels) or empty + hint |
| Catalog | `ResultLane` of plans from GQL `plans` (backend `PlanBridge` returns **ACTIVE** only — website query has no status filter); card CTA **اشتراك** / disabled **الحالية** |
| Checkout | Hides other plans; selected plan; **نظام الاشتراك** (`MONTHLY` / `YEARLY`); methods list; cancel |
| Pay | Inline on selected method row (width/gap transition); disabled until method + `canSubscribe === CAN` + form ready; no bottom CTA |

## 4) Data hooks (GQL)

| Hook | Private adapter id | Notes |
|---|---|---|
| `useCustomerSubscriptionPlans` | `"customer-subscription-plans"` | Inherit `CUSTOMER_GQL`, `listable: "plans"`. Exposes `isInitialLoading` / `loadMore` / `totalCount` (`_Plan.total_count`) |
| `useCustomerSubscriptionPaymentMethods` | `"customer-subscription-payment-methods"` | Inherit `CUSTOMER_GQL`, `listable: "subscriptionPaymentMethods"`. Idle when `!planId` |
| `useCustomerSubscriptionCanSubscribe` | `"customer-subscription-can-subscribe"` | Inherit `CUSTOMER_GQL`; reads `section.me.canSubscribe` (not a listable). Idle when `!planId` |

### 4.1 Payment methods are not paginated

`_PaymentMethod` is **not** `_Pagination` and has **no** `total_count` on the website GQL mirror. Do **not** invent `totalCount` / `loadMore` / `pageSize` on the payment-methods hook. Plans keep pagination because `_Plan` implements `_Pagination`.

### 4.2 Loading flags (do not conflate)

| Hook | Flag | True when |
|---|---|---|
| Plans | `isInitialLoading` | `!exist \|\| LOADING \|\| INIT` (catalog `ResultLane`) |
| Payment methods | `isLoading` | `!exist \|\| LOADING \|\| INIT \|\| RELOADING` |
| canSubscribe | `isLoading` | same as payment methods |

Plan/period changes call `mLoad({ reload: true })` → adapter `RELOADING`. Method skeletons (and canSubscribe gate) must treat `RELOADING` as loading — do not rename methods/ability flags back to `isInitialLoading`.

## 5) Subscribe form

| Item | Contract |
|---|---|
| Identity | `Forms.CUSTOMER_SUBSCRIPTION` on the screen (`useShallowForm`) — no separate form hook file |
| Sub | `subscribe` |
| Values | `plan`, `plan_billing_period` (`MONTHLY`\|`YEARLY`), `method_id` (number) |
| Success | `autoMainMessage: false`; read `paymentURL` from `res.data.other`; `location.assign` if non-empty string; else stay and allow retry |
| Double-submit | `submittingRef` + `sending` (`status === "SENDING"`) |
| Ability | Gate Pay with `me.canSubscribe` → `Able.value === "CAN"`; show `description` when denied |

## 6) `paymentReturn` UX

Finalize (backend) 302 → `{WEBSITE_URL}/customer/subscription?paymentReturn=success|error` (`MyFatoorahFinalizeController.sendWebsitePaymentRedirect`).

On this screen only (ref-guarded once):

1. `toast.showToast({ status: SUCCESS|ERROR, description })` from page i18n (manual toast — not a form main-message).
2. `useMe().update()`.
3. `nav.push({ identify: "CustomerSubscription", replace: true })` — omit query so refresh does not re-toast.

## 7) Money UI (SAR)

| Surface | Contract |
|---|---|
| `SarMark` / `SarAmount` | Shared under `components/` (SAMA U+20C1 paths). Not under `customer/`. |
| Plan card + current card | Always render amounts through `SarAmount` (catalog prices are SAR). |
| Method row | `(currencyIso \|\| "SAR")` empty/SAR → `SarAmount`; else `"${amount} ${ISO}"`. |
| Forbidden for SAR | Legacy abbreviation `ر.س` / plain `SAR` text as the amount chrome. |

Governance: `.cursor/rules/website-customer-utils-composed-marks.mdc`.

## 8) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.subscription.*` | Page titles, catalog, period, methods, CTAs, paymentReturn, current-card relative-end labels |
| `ui.pages.customer.meetingDetails.scheduleNoteSubscriptionRequired` / `…Missing` | Meeting draft schedule notes (owned by meetings screen) |
| `ui.components.customerDrawer.itemSubscription` | Drawer tile |

Presentational cards take labels as props (W42) — translators stay on the screen / drawer / meetings screen.

## 9) Failure / empty modes

| Mode | Behavior |
|---|---|
| No plans | ResultLane empty (backend ACTIVE catalog empty) |
| Methods empty / error | Empty + `LaneFailed` retry |
| Methods loading / reloading | Method-row skeletons (`paymentMethods.isLoading`) |
| `canSubscribe` not `CAN` | Pay disabled; show description when present |
| Missing `paymentURL` | Stay; retry allowed |
| Finalize error | `paymentReturn=error` toast |

## 10) Related customer surfaces (not owned by this page)

| Surface | Behavior | Doc |
|---|---|---|
| `useMe` | `coreQuery` nests `currentSubscription` (+ plan) | `flow-customer-shell.md` |
| Customer home | Hero subscription label; attention `no_subscription` when missing (**no** `href`) | `flow-customer-shell.md` §7 |
| Meeting details | Draft **schedule notes** for subscription required/missing — **not** a readiness row | `flow-customer-meetings.md` §6.5 |

## 11) Active-subscription gates (backend trust)

Helper: `Customer.getCurrentSubscription` (association `currentSubscription` / scope `current`). Denial key: `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` (“An active subscription is required.” / «يلزم وجود اشتراك فعّال.»).

| Gate | Where | Contract |
|---|---|---|
| Meeting approve | `Customer.can` `MEETING` / `approve` | `meeting-domain.md` §9.1b — draft create/update still allowed without a subscription |
| Meeting socket handshake | `MeetingAuthenticationIOMiddleware` | `meeting-domain.md` §9.1c; `meeting-realtime-socket.md` §2 / §6.1 — website linking `FAILED` (session `NOT_VALID` today) |
| LiveKit join token | `OrganizationHostMiddleware` (`org_host`) | `meeting-domain.md` §9.1c; `livekit-media-plane.md` §7.2; `client-portal-http-website.md` — `/custom/org/start` does **not** use `org_host` |

## 12) Traceability map

### Website (`website/`)

| Path | Role | Section |
|---|---|---|
| `src/app/ui/pages/customer/CustomerSubscription.tsx` | Thin page | §2 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionScreen.tsx` | Screen + subscribe + paymentReturn | §3–§6 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionCurrentCard.tsx` | Current entitlement card | §3, §7 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionPlanCard.tsx` | Plan card CTAs + SAR | §3, §7 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionPlanCardSkeleton.tsx` | Plan skeleton | §3 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionMethodRow.tsx` | Method + inline Pay + SAR/ISO | §3, §7 |
| `src/app/ui/components/customer/subscription/CustomerSubscriptionMethodRowSkeleton.tsx` | Method skeleton | §4.2 |
| `src/app/ui/components/customer/hooks/useCustomerSubscriptionPlans.ts` | Plans adapter | §4 |
| `src/app/ui/components/customer/hooks/useCustomerSubscriptionPaymentMethods.ts` | Methods adapter + `isLoading` | §4.1–§4.2 |
| `src/app/ui/components/customer/hooks/useCustomerSubscriptionCanSubscribe.ts` | Ability adapter + `isLoading` | §4, §5 |
| `src/app/ui/components/SarMark.tsx` | SAMA glyph | §7 |
| `src/app/ui/components/SarAmount.tsx` | Amount + mark | §7 |
| `src/app/ui/components/ResultLane.tsx` | Optional `skeletonCount` (default 6) | §3 |
| `src/app/ui/components/customer/meetings/CustomerMeetingDetailsScreen.tsx` | Draft schedule subscription notes | §10 |
| `src/app/ui/components/customer/CustomerDrawer.tsx` | Tile identify | §2 |
| `src/app/ui/components/customer/hooks/useMe.tsx` | `currentSubscription` | §10 |
| `src/app/ui/components/customer/hooks/useCustomerHome.ts` | `hasSubscription` / `no_subscription` attention | §10 |
| `src/app/ui/components/customer/home/CustomerHomeScreen.tsx` | Hero / attention chrome | §10 |
| `src/resources/configs/routes.ts` | Route + `paymentReturn` query | §2 |
| `src/resources/configs/store/forms.ts` | `CUSTOMER_SUBSCRIPTION` | §5 |
| `src/types/requesters/requesters.website.ts` | `customer.subscription: "subscribe"` | §5 |
| `src/resources/translations/ar.ts` / `en.ts` | Page + meeting schedule-note keys | §8 |
| `src/types/gql/**` | Prior Plan/Subscription GQL mirror | `data-flow-and-gql.md` |
| `lib/tsconfig.tsbuildinfo` | Generated — not narrated | — |

### Backend (`backend/`)

| Path | Role | Section |
|---|---|---|
| `src/app/http/controllers/external/payments/MyFatoorahFinalizeController.ts` | 302 → `/customer/subscription?paymentReturn=` | §6 |
| `src/app/orm/models/Customer.ts` | `approve` requires current subscription | §11 |
| `src/app/http/middlewares/OrganizationHostMiddleware.ts` | `org_host` subscription gate | §11 |
| `src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | Handshake subscription gate | §11 |
| `src/resources/trans/ar/messages.ts` / `en/messages.ts` | `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED` | §11 |
| `src/app/gql/bridges/customer/PlanBridge.ts` | ACTIVE-only catalog | §3; `plan-domain.md` |

### Docs / governance (root)

| Path | Role |
|---|---|
| `docs/platforms/website/flow-customer-subscription.md` | This page |
| `docs/platforms/website/flow-customer-meetings.md` | Schedule notes + approve UX (§6.2 / §6.5) |
| `docs/platforms/website/data-flow-and-gql.md` | Adapter ids |
| `docs/platforms/website/flow-form-foundation.md` | Form registry |
| `docs/platforms/website/flow-customer-shell.md` | Drawer / home |
| `docs/platforms/website/route-registry-contract.md` | Route shipped |
| `docs/platforms/website/component-structure.md` | Folder inventory |
| `docs/platforms/website/overview.md` / `README.md` | Indexes + traceability |
| `docs/platforms/backend/contracts/meeting-domain.md` | §9.1b approve; §9.1c live entry |
| `docs/platforms/backend/contracts/meeting-realtime-socket.md` | Handshake |
| `docs/platforms/backend/contracts/client-portal-http-website.md` | `org_host` |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | Token errors |
| `docs/platforms/backend/contracts/http-and-requesters.md` | `org_host` note |
| `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md` | Finalize redirect |
| `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md` | Finalize redirect |
| `docs/platforms/website/organization-host-routing.md` | Host + handshake + org_host |
| `.cursor/rules/website-customer-utils-composed-marks.mdc` | SarMark / SarAmount |
| `.cursor/rules/meeting-lifecycle-approve-lock.mdc` | Approve subscription |
| `.cursor/rules/meeting-realtime-socket.mdc` | Handshake subscription |
| `.cursor/rules/livekit-media-plane.mdc` | org_host subscription |
| `.cursor/rules/organization-host-routing.mdc` | org_host pairing |
| `.cursor/skills/website-customer-subscription/SKILL.md` | Repeatable shipping workflow |

## 13) Related

- `flow-customer-shell.md`
- `flow-customer-meetings.md` §6.5
- `flow-form-foundation.md`
- `route-registry-contract.md` §5.2
- `data-flow-and-gql.md`
- `organization-host-routing.md` §6.2–§6.3, §8
- `docs/platforms/backend/contracts/plan-domain.md`
- `docs/platforms/backend/contracts/subscription-domain.md`
- `docs/platforms/backend/contracts/myfatoorah-invoice-payment-domain.md`
- `docs/platforms/backend/contracts/external-http-mount-and-myfatoorah-callbacks.md`
- `docs/platforms/backend/contracts/meeting-domain.md` §9.1b–§9.1c
