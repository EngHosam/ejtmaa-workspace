# Website GraphQL Mirror and Tooling

Customer portal contract (see `overview.md`).

## Purpose

GraphQL mirror and tooling contract for `website/`:

- backend source-of-truth ownership,
- mirrored SDL and generated type placement,
- local GraphQL tooling config placement,
- verification expectations.

## 1) Source of truth

Backend owns the website GraphQL contract:

- `backend/src/app/gql/definitions/base.graphql`
- `backend/src/app/gql/definitions/customer.graphql`
- `backend/src/app/gql/gql-types/base.ts`
- `backend/src/app/gql/gql-types/customer.ts`

`website/` consumes copied mirrors of those files.

## 2) Website mirror surfaces

### SDL mirrors

- `website/src/types/gql/definitions/base.graphql`
- `website/src/types/gql/definitions/customer.graphql`
- `website/src/types/gql/definitions/shared.graphql` — shared `Query { notifications }` extension stub

Mirror artifacts only — not website-owned schema authoring surfaces.

Customer mirrors currently include organization, member (+ `_MemberFilter`), message-template, meeting, meeting-participant, agenda-item, decision, vote, talk-record, plan catalog, subscription reads, subscription payment-method reads, and Me ability gates from backend contracts (`organization-domain.md`, `member-domain.md`, `message-template-domain.md`, `meeting-domain.md`, `meeting-participant-domain.md`, `agenda-item-domain.md`, `decision-domain.md`, `vote-domain.md`, `talk-record-domain.md`, `plan-domain.md`, `subscription-domain.md`, `myfatoorah-invoice-payment-domain.md`): roots `members(filter: _MemberFilter)`, `member(id)`, `messageTemplates`, `messageTemplate(id)`, `meetings`, `meeting(id)`, `plans`, `plan(id)`, `subscriptions`, `subscription(id)`, `subscriptionPaymentMethods(planId, billingPeriod)`; types `_Organization` / `_Member` / `_MemberFilter` / `_MessageTemplate` / `_Meeting` / `_MeetingParticipant` / `_AgendaItem` / `_Decision` / `_Vote` / `_TalkRecord` / `_Plan` / `_Subscription` / `_PaymentMethod` (nested `_Me.organization`, `_Me.currentSubscription`, `_Me.canDeleteNotifications`, `_Me.canSubscribe(planId)`, `_Subscription.plan`, `_Meeting.participants` + `_MeetingParticipant.member`, `_Meeting.agendaItems`, `_Meeting.decisions`, `_Decision.votes` + `_Vote.member`, `_Meeting.talkRecords` + `_TalkRecord.member`; **no** customer root `organization`; no root participant/agenda/decision/vote/talkRecord/invoice queries; plan roots are public ACTIVE catalog; subscriptions are me-scoped; payment methods are gateway-derived for ACTIVE plan price). Requester mirror includes `customer.subscription: "subscribe"`. Website members directory consumer: `flow-customer-members.md`; org settings + setup gate: `flow-customer-organization.md`.

### Generated type mirrors

- `website/src/types/gql/gql-types/base.ts`
- `website/src/types/gql/gql-types/customer.ts`

Copied/generated contract artifacts — do not hand-edit.

## 3) Local GraphQL tooling scope

Root schema project:

- `website/graphql.config.yml` — project pointing at mirrored `base` + `customer` under `src/types/gql/definitions/` (schema list: `base.graphql` + `customer.graphql`).

Subtree helper configs (when GraphQL-aware UI code exists in that subtree):

- `website/src/app/ui/pages/graphql.config.yml`
- `website/src/app/ui/components/graphql.config.yml`

Shipped helper configs point at `base.graphql` only (subtree editor awareness); they do not declare independent schemas. `src/app/ui/pages/customer/graphql.config.yml` and `src/app/ui/components/customer/graphql.config.yml` are placeholder configs for future customer screens.

## 4) Copy workflow

1. Backend files remain source of truth.
2. Website mirrors sync by command-based copy (per `.cursor/rules/gql-schemas-bridges-general.mdc`).
3. Copied SDL and gql-types are not hand-edited in `website/`.
4. Add local `graphql.config.yml` only in UI subtrees that host GraphQL-aware code.

## 5) Role boundary

- `website/` mirrors `base` + `customer` only.
- `supervisor` mirrors belong to `cpanel/` only.

## 6) Consumption

Website reads use `DATA_ADAPTERS.GQL` against the customer schema. See `data-flow-and-gql.md`.

## 7) Verification

- `yarn run type-check` (website)
- After backend SDL changes: `yarn generate-types` (backend), then sync mirrors to `website/` and `cpanel/` per `.cursor/rules/gql-schemas-bridges-general.mdc`

## Related

- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/backend/contracts/graphql-and-types.md`
- `.cursor/rules/gql-schemas-bridges-general.mdc`
