# CPanel Supervisor Settings

Supervisor account settings (name / optional password pair). Email is displayed read-only and is **not** submitted. Identify `SupervisorSettings` at `/supervisor/settings`. Overlay: `supervisor-state-lanes.md`.

## 1) Form

`Forms.SUPERVISOR_SETTINGS` → `API.FORMS.SUPERVISOR.R("supervisor")`.

`useSupervisorSettings`: inherit form identify, `didEntered` → `sub: "read"`. Save → `sub: "update"` with body `{ name, current_password?, new_password? }`. Both password fields omitted when either is empty; both required together when changing password. After success, password fields are cleared locally (no navigation).

`FormTextField` `email` is `disabled`. `FormStateLane` wraps fields only; title + save stay outside. Save disabled while loading/empty/failed.

Drawer: `FiSettings`, `alwaysAvailable`. Support tile (`identify: "SupervisorSupport"`) has no `alwaysAvailable` and is **not** in `routes.ts`, so `isAvailable` is false: the tile still renders, click is a no-op. Not a page.

## 2) Backend

`SupervisorRequester` `update`: `startTransaction()` (no `"join_current"`). Updates supervisor + user `name`. If both passwords present: compare current, set new password, `Token.destroy` all other tokens for that user (`force: true`). `transaction.afterCommit` → `notify("OnSupervisorEvent", { type: "UPDATED", supervisor })`.

`OnSupervisorEvent.handle()` is empty (no in-app `Notification` row). Broadcast: namespace supervisor, room `Rooms.SUPERVISOR(id)`, payload `{ type: "UPDATED", ...__global__(supervisor) }`.

Cpanel `useMe` `useSocket({ registerTo: "OnSupervisorEvent" })` reloads `me { id name email }`. Registry: `cpanel/src/resources/configs/socket/events.ts` + `cpanel/src/types/events.ts`.

Visitor login stays on `VisitorRequesterController` (`POST /cpanel/forms/requester/...`, `Authed()`). Supervisor forms: `POST /cpanel/forms/supervisor/requester/...`.

## 3) Traceability

| Path | Role |
|---|---|
| `cpanel/src/app/ui/components/supervisor/settings/useSupervisorSettings.ts` | Form hook |
| `cpanel/src/app/ui/components/supervisor/settings/SupervisorSettingsScreen.tsx` | Screen |
| `cpanel/src/app/ui/pages/supervisor/SupervisorSettings.tsx` | Thin page |
| `cpanel/src/resources/configs/store/forms.ts` | `SUPERVISOR_SETTINGS` |
| `cpanel/src/app/ui/components/supervisor/hooks/useMe.tsx` | Socket reload |
| `cpanel/src/resources/configs/socket/events.ts` | Event stub |
| `cpanel/src/types/events.ts` | `OnSupervisorEventDate` |
| `backend/src/app/orchestrator/requesters/SupervisorRequester.ts` | read/update + notify |
| `backend/src/app/notify/events/OnSupervisorEvent.ts` | Broadcast |
| `backend/src/app/types/Events.ts` | Payload type |
| `backend/src/app/http/controllers/cpanel/forms/VisitorRequesterController.ts` | Visitor forms |
| `backend/src/app/http/routes/cpanel.ts` | Split form routers |

## Related

- `docs/platforms/backend/contracts/socket-event-mirroring.md`
- `docs/platforms/cpanel/supervisor-state-lanes.md`
- `docs/platforms/cpanel/supervisor-shared-ui.md`
- `.cursor/rules/socket-event-mirroring.mdc`
