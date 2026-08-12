---
name: website-customer-subscription
description: >-
  Ships or extends the website customer Plan catalog and Subscription checkout
  (CustomerSubscription route/screen, Forms.CUSTOMER_SUBSCRIPTION subscribe +
  paymentURL redirect, paymentReturn toast, SarMark/SarAmount, non-paginated
  payment methods isLoading including RELOADING) and keeps active-subscription
  gates aligned (approve, Meeting handshake, LiveKit org_host). Use when changing
  subscription UI, checkout form, payment-methods adapters, finalize redirect, or
  subscription-gated meeting/live entry.
---

# Website customer subscription

## When to Use

- Changing `CustomerSubscription` route/page/screen, plan/method cards, or skeletons.
- Changing `useCustomerSubscriptionPlans` / `PaymentMethods` / `CanSubscribe`.
- Changing `Forms.CUSTOMER_SUBSCRIPTION` subscribe → `paymentURL` / `paymentReturn`.
- Changing `SarMark` / `SarAmount` money chrome used by checkout.
- Changing finalize redirect to `/customer/subscription`, or subscription gates on approve / `meeting_auth` / `org_host`.

## Do Not Use For

- Plan/Subscription ORM or MyFatoorah invoice domain internals alone — backend plan/subscription/myfatoorah contracts.
- Meeting shell broadcast UI (except the subscription gate) — `website-meeting-shell` / `meeting-livekit-token`.

## Instructions

1. Read `docs/platforms/website/flow-customer-subscription.md` (authoritative website contract) and §11 for backend gates.
2. Keep product names: **Plan** = الباقة, **Subscription** = الاشتراك. Never introduce `Package` for this domain.
3. Screen owns zones + `useShallowForm(Forms.CUSTOMER_SUBSCRIPTION)` — no separate form hook file. Success: `autoMainMessage: false` → `res.data.other.paymentURL` → `window.location.assign`.
4. `paymentReturn` is owned **only** by `CustomerSubscriptionScreen` (toast + `useMe().update` + replace without query). Do not handle it on Home.
5. Payment methods: **not** `_Pagination`. Public flag is `isLoading` and **must** include `RELOADING` (plan/period reload). Do not invent `totalCount` / `loadMore`. `canSubscribe` uses the same `isLoading` shape. Do not rename those flags to `isInitialLoading`.
6. Plans keep `_Pagination` / `total_count` / load-more and catalog `isInitialLoading` (LOADING/INIT only). Catalog uses `ResultLane` with `skeletonCount={3}`. ACTIVE filtering is **PlanBridge** — do not invent a website status filter on the plans query.
7. SAR: plan + current cards always use `SarAmount`; method row uses SAR/`SarAmount` when ISO empty or SAR, else `"${amount} ${ISO}"`. No `ر.س` / plain `SAR` text chrome. Rule: `.cursor/rules/website-customer-utils-composed-marks.mdc`.
8. Drawer tile unlock = identify registered in `routes.ts` (`CustomerDrawer` already has `CustomerSubscription` / `FiCreditCard`).
9. Meeting details: subscription hint = draft **schedule notes** only — never a readiness checklist row; never under the Approve button. Approve Ability still enforces `getCurrentSubscription` (`meeting-domain.md` §9.1b).
10. Active subscription gates (same helper, same message key `MEETING_ACTIVE_SUBSCRIPTION_REQUIRED`):
    - Approve: `Customer.can` `MEETING`/`approve` (§9.1b)
    - Live entry: `MeetingAuthenticationIOMiddleware` + `OrganizationHostMiddleware` / `org_host` (§9.1c)
    Do not put the gate on `/custom/org/start`. Do not document live gates as approve-completeness steps.
11. Presentational labels: W42 — cards take props; translators on screen/drawer.
12. Verify with existing scripts only: `yarn type-check` in `website/` and `backend/` when those repos change.
