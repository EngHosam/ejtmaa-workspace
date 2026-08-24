# CPanel Supervisor Shared UI (this wave)

Shared `cpanel/src/app/ui/components/` pieces added or changed with the supervisor workspace modules. Overlay hosts live in `supervisor-state-lanes.md`.

## 1) `StatusChip`

`cpanel/src/app/ui/components/StatusChip.tsx`

Pill: optional Feather icon + caption. Empty label → `null`. Tones: `neutral` | `info` | `warning` | `success` | `danger` mapped to `semanticColor.state*` / `sectionBrandBackground` (not chart brand fills).

Helpers (GQL enum **values**, labels stay from the server):

| Helper | Values |
|---|---|
| `meetingStatusVisual` | DRAFT neutral, WAITING_TO_START warning, STARTED success, COMPLETED info, CANCELED danger |
| `meetingNotifyVisual` | WAITING_TO_NOTIFY warning, NOTIFIED success, PARTIALLY_NOTIFIED warning, FAILED danger |
| `organizationStatusVisual` | ACTIVE success, DISABLED danger |
| `subscriptionStatusVisual` | ACTIVE success, EXPIRED danger, REPLACED warning |

Used on meeting cards/detail, subscription cards/detail, customer org status, home meeting cards. Do not invent a second pill.

## 2) `FormMultiLangField`

`cpanel/src/app/ui/components/form/FormMultiLangField.tsx`

Two nested inputs (`ar` / `en`) under one form `name`. Nested errors: `errors[name.ar]` / `errors[name.en]`. Setting both languages to blank stores `null` (matches optional `isMultiLang` on the backend). `multiline` → textarea rows 5.

Shipped consumer: plan `name` / `description`.

## 3) `FormTextField.disabled`

Optional `disabled` forces read-only (no `onChange`) and passes through `FormInputWrapper`. Settings uses it on `email` (email is **not** in the supervisor update payload).

## 4) `Stats` `href` / `note`

`StatItem` may include `note` (secondary caption) and `href` (`Link` wrapping the tile). Home uses both. Directory headers currently use values only.

## 5) Overlay busy flag (detail hooks)

`isInitialLoading` is `!exist || !status || (isBusy && !record)` where `isBusy` is `LOADING` | `RELOADING` | `INIT`. Reload **with** a record does not replace the pane with `Loadable`. Same pattern on customer, meeting, and subscription detail hooks.

List hooks use `isBusy` to block load-more, not to hide the grid.

## 6) Traceability

| Path | Role |
|---|---|
| `cpanel/src/app/ui/components/StatusChip.tsx` | §1 |
| `cpanel/src/app/ui/components/form/FormMultiLangField.tsx` | §2 |
| `cpanel/src/app/ui/components/form/FormTextField.tsx` | §3 |
| `cpanel/src/app/ui/components/Stats.tsx` | §4 |
| `cpanel/src/app/ui/components/PageStateLane.tsx` | padding/overlay host; `supervisor-state-lanes.md` |
| `cpanel/src/app/ui/components/FormStateLane.tsx` | alias |

## Related

- `docs/platforms/cpanel/supervisor-state-lanes.md`
- `docs/platforms/cpanel/plan-management.md`
- `docs/platforms/cpanel/supervisor-home.md`
