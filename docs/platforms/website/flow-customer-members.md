# Website Flow — Customer Members (Directory + Form)

Authenticated customer org-member directory and create/edit/delete form on `CUSTOMER_MAIN`. Shell/breadcrumb contract remains in `flow-customer-shell.md` §7.1. Backend write contract: `docs/platforms/backend/contracts/member-domain.md` §9. Form foundation: `flow-form-foundation.md`.

## 1) Scope

**Shipped**

- Member directory for the authenticated customer's organization (GQL list + server search + ResultLane load-more).
- Route-query persistence under history key `members` (`useWithHistoryState`); Enter-to-commit search.
- Multi-path form route `CustomerMemberForm` (`create` / `update`) with `MemberRequester` subs `read` | `create` | `update` | `delete`.
- Fields: name (required), email / mobile (optional, with meeting-link subtitles), avatar (`avatar_file` via immediate multipart upload).
- List Add → create form; card Edit → update form; form Save / Delete; `SectionHeading` optional back icon → `nav.back()`.
- Automatic success toasts (W11); no manual `showToast(mainMessage)`.

**Not shipped**

- GQL `can*` UI gating (Add/Edit always visible).
- Soft-delete / cascade delete when roster or chairperson refs exist (backend **blocks** with specific message keys).
- Distinct empty copy for “no search hits” vs “no members yet”.
- Page / sort / scope route params (load-more list only).
- Settings page reuse of `FormAvatarField` (field is shared-ready; settings flow still planned).

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| List route | `CustomerMembers` → `/customer/members` |
| Form route | `CustomerMemberForm` → `path.create` `/customer/members/form`, `path.update` `/customer/members/form/:id` |
| List page | `website/src/app/ui/pages/customer/CustomerMembers.tsx` |
| Form page | `website/src/app/ui/pages/customer/CustomerMemberForm.tsx` |
| List screen | `…/members/CustomerMembersScreen.tsx` |
| Form screen | `…/members/CustomerMemberFormScreen.tsx` |
| Hook | `…/hooks/useCustomerMembers.ts` |
| Card | `…/members/CustomerMemberCard.tsx` |
| Href builder | `website/src/resources/configs/customer/formRoute.ts` → `buildCustomerMemberFormHref` |

Both list and form: `layout: "CUSTOMER_MAIN"`, `mustAuthedAs: ["CUSTOMER"]`. Form breadcrumb parent `CustomerMembers`; label switches on `params.id` (create vs edit title).

## 3) Directory data adapter

| Concern | Value |
|---|---|
| Mount-private adapter id | `"customer-members"` |
| Inherited | `DATA_ADAPTERS.CUSTOMER_GQL` → `API.DATA_ADAPTERS.CUSTOMER.GQL` |
| Listable | `"members"` |
| Default page size | `initDataAdaptersProps.default.maxLoadLength` = **24** |
| Reload | `mLoad({ reload: true, query })` when committed history query changes; same query on load-more / refresh |

GQL (inline in hook):

```graphql
query CustomerMembers($filter: _MemberFilter) {
    members(filter: $filter) {
        id
        name
        email
        mobile
        avatar_url
        total_count
    }
}
```

History search: `.cursor/rules/website-customer-list-history-search.mdc`. Backend filter: `member-domain.md` §4–§5.

## 4) Directory screen composition

`Container` → `Col pt={2} gap={1.5} pb={2}`:

1. Row `jc_sb`: `SectionHeading` + primary `FormActionButton` Add → `nav.push(buildCustomerMemberFormHref("create"))`.
2. `SearchField` (draft + Enter commit).
3. `ResultLane` → `CustomerMemberCard` (`onEdit` → `buildCustomerMemberFormHref("update", member.id)`).

Shared ResultLane chrome: §6 of the prior directory slice remains (`ResultLane`, `CardSkeleton` member-shaped, `SearchField`, `Wrong`). Skeleton rule: `.cursor/rules/website-result-lane-skeleton-shape.mdc`.

## 5) Form route + `Forms.CUSTOMER_MEMBER`

| Concern | Value |
|---|---|
| Form identity | `Forms.CUSTOMER_MEMBER` → `api: API.FORMS.CUSTOMER.R("member")` |
| Multi-path | One identify; `isUpdate = !!id` from `useCurrentParams<"CustomerMemberForm">` — never branch on page identify |
| Create reducer key | Stable `"customer-member-form-create"` + `removeOnExit: false` |
| Update | Auto identify; `removeOnExit: true`; `initProps.values = { member: id }`; `didEntered` → `send({ sub: "read" })` |
| Loading | `Loadable` while `!exist` or update `read` in flight (`INIT` / `SENDING` + `currentSub === "read"`) |

Governance: `.cursor/rules/website-multi-path-form-routes.mdc`, `.cursor/rules/website-shallow-form-submit-and-cleanup.mdc`, `.cursor/rules/website-form-success-toast-automatic.mdc`.

### 5.1 Screen chrome

- Header row: `SectionHeading` (`onBack` → `nav.back()`, `backLabel` from `ui.components.mainHeader.back`) + action buttons (not full-width under fields).
- Actions: primary Save/Add (`sub: create|update`, `afterSentSuccess: nav.back()`); update-only neutral Delete (`window.confirm` → `sub: "delete"` → `nav.back()`). Guard with `submittingRef`.
- Fields column `maxW={32}`: `FormAvatarField` + `FormTextField` name / email / mobile.
- Email / mobile `subTitle`: meeting-link purpose copy (i18n). No placeholders unless product asks.
- Avatar preview after `read`: form value `avatar_file` resolved via `mediaImageUri` (no separate `currentAvatarUrl` required).

### 5.2 Avatar upload

`FormAvatarField` → `uploadEntityMediaFile` → `API.ACTIONS.MULTIPART_UPLOAD` → store `uploadedName` in form field `avatar_file`. Helpers: `website/src/app/helpers/entityMedia.ts`, `media.ts` (`mediaStaticRoot` / `mediaImageUri`). Visual language: navy identity circle + quiet text action (Ejtmaa), not Masdaria camera-chip chrome. Rule: `.cursor/rules/website-form-avatar-field.mdc`.

## 6) Shared field contracts (touched this slice)

| Surface | Contract |
|---|---|
| `FormTextField` | Inherit page direction; `textAlign: "start"`; **do not** force `ltr` / `ta_l` (caret bug in Arabic). Rule: `.cursor/rules/website-form-text-field-direction.mdc` |
| `FormInputWrapper` `subTitle` | `variant="caption"` + `semanticColor.textTertiary` (quieter than `inputTitle`) |
| `SectionHeading` | Optional `onBack` + `backLabel`; back control uses `FiArrowLeft` + `flp`; row uses `flx_1 minW={0}` so sibling header actions can sit `jc_sb` |
| `FormActionButton` | Header actions without `fullWidth` on this form |

## 7) i18n

| Key path | Purpose |
|---|---|
| `ui.pages.customer.members.*` | Directory title, search, empty, Add, Edit, load-more |
| `ui.pages.customer.memberForm.*` | create/edit titles, subtitle, avatar labels, field labels, email/mobile subtitles, create/update/delete + confirm |
| `ui.components.mainHeader.back` | SectionHeading back aria/title on form |
| `ui.pages.uiMockup.formAvatarReview.*` / `sections.formAvatar` | UiMockup avatar preview |
| `ui.pages.uiMockup.sectionHeadingReview.withBack` | UiMockup back-icon preview |

ar/en mirrors required.

## 8) Failure / empty modes (UI)

| Condition | UI |
|---|---|
| Directory initial load | `CardSkeleton` grid |
| Directory empty | `Empty` overlay |
| Directory fail | `LaneFailed` + retry |
| Form update read | Centered `Loadable` |
| Delete blocked (roster/chairperson) | Backend main message toast (automatic); form stays |
| Validation errors | Field errors via form reducer |

## 9) Traceability map (this change set)

### Backend (`backend/` repo)

| Path | Status | Doc |
|---|---|---|
| `src/app/orm/models/Customer.ts` | `Ability.MEMBER` + `can` | `member-domain.md` §9.1 |
| `src/app/validation/joi_rules.ts` | `isCustomerOwnedMember` | §9.2 |
| `src/app/orchestrator/requesters/MemberRequester.ts` | added | §9.3 |
| `requesters.website.ts` | `customer.member` | §9.4 |
| `src/resources/trans/ar/messages.ts` / `en/messages.ts` | delete + success keys | §9.4 |

### Website (`website/` repo)

| Path | Status | Doc |
|---|---|---|
| `src/types/requesters/requesters.website.ts` | `customer.member` | this §5; `flow-form-foundation.md` |
| `src/resources/configs/store/forms.ts` | `Forms.CUSTOMER_MEMBER` | §5 |
| `src/resources/configs/customer/formRoute.ts` | href builder | §2, §5 |
| `src/resources/configs/routes.ts` | `CustomerMemberForm` multi-path | §2; `route-registry-contract.md` |
| `src/app/ui/pages/customer/CustomerMemberForm.tsx` | thin page | §2 |
| `src/app/ui/components/customer/members/CustomerMemberFormScreen.tsx` | form screen | §5 |
| `src/app/ui/components/customer/members/CustomerMembersScreen.tsx` | Add wiring | §4 |
| `src/app/ui/components/customer/members/CustomerMemberCard.tsx` | `onEdit` | §4 |
| `src/app/ui/components/form/FormAvatarField.tsx` | avatar field | §5.2 |
| `src/app/helpers/entityMedia.ts` | multipart upload | §5.2 |
| `src/app/helpers/media.ts` | static root + image URI | §5.2 |
| `src/app/ui/components/form/FormTextField.tsx` | direction fix | §6 |
| `src/app/ui/components/form/FormInputWrapper.tsx` | subtitle tertiary | §6 |
| `src/app/ui/components/SectionHeading.tsx` | optional back | §5.1, §6 |
| `src/app/ui/pages/UiMockup.tsx` | avatar + heading back previews | §7 |
| `src/resources/translations/ar.ts` / `en.ts` | members + memberForm + mockup | §7 |
| `src/app/ui/components/customer/hooks/useCustomerMembers.ts` | unchanged list hook | §3 |
| `lib/tsconfig.tsbuildinfo` | generated | skip narrative |

### Root docs / governance

| Path | Role |
|---|---|
| `docs/platforms/website/flow-customer-members.md` | this flow |
| `docs/platforms/website/flow-form-foundation.md` | forms registry + field surfaces |
| `docs/platforms/backend/contracts/member-domain.md` | write surface §9 |
| `docs/platforms/website/route-registry-contract.md` | `CustomerMemberForm` |
| `docs/platforms/website/overview.md` | index |
| `docs/platforms/website/data-flow-and-gql.md` | requester map |
| `docs/platforms/website/flow-customer-shell.md` | shell shipped note |
| `.cursor/rules/website-form-avatar-field.mdc` | avatar field |
| `.cursor/rules/website-form-text-field-direction.mdc` | input direction |
| `.cursor/skills/website-customer-member-form/SKILL.md` | repeatable form workflow |

## 10) Related

- `docs/platforms/website/flow-customer-shell.md`
- `docs/platforms/website/flow-form-foundation.md`
- `docs/platforms/website/data-flow-and-gql.md`
- `docs/platforms/website/route-registry-contract.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `.cursor/rules/website-multi-path-form-routes.mdc`
- `.cursor/rules/website-customer-list-history-search.mdc`
- `.cursor/rules/website-result-lane-skeleton-shape.mdc`
- `.cursor/skills/website-customer-result-lane-list/SKILL.md`
- `.cursor/skills/website-customer-breadcrumb-subpage/SKILL.md`
