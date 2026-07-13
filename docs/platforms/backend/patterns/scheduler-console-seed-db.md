# Scheduler, Console, Seed, and DB Operations (Ejtmaa)

## 1) Scheduler

Config: `backend/src/resources/configs/scheduler/index.ts`, `backend/src/resources/configs/scheduler/cron.ts`

Current state:
- Example task `TestTask` registered in `cron.ts` for framework wiring
- Production domain scheduler jobs land as modules are added

Adding a task:
1. Create class under `backend/src/app/scheduler/tasks/` extending `TaskBase`
2. Register in `cron.ts` with static import
3. Ensure idempotency and observability

## 2) Console

Registry: `backend/src/console/Console.ts`

Sub consoles:
- `DatabaseConsole` — init, alter, update
- `EjtmaaConsole` — settings summary, clear notifications
- `UtilsConsole` — utilities

### DatabaseConsole flows

- `init` — install, sync, routines, seed, post
- `alter` — schema alter + patches
- `update` — targeted seed/maintenance actions via `SeedPatch.update(action)`

## 3) Seed Patch

File: `backend/src/app/orm/patches/SeedPatch.ts`

Current `init()` seeds:
- system settings defaults (about, terms, privacy, seller_name = "Ejtmaa", VAT flags)
- default supervisor admin account when none exists
- demo customers via `seedDemoCustomers` (extension hook for demo rows)

`SeedPatch.update(version)` is reserved for future targeted maintenance actions. The checked-in switch currently rejects unknown versions with `NOT_VALID`.

## 4) Safety rules

- Jobs and seed actions must be idempotent
- Use transactions for multi-step writes
- Observable logs for critical branches
- Do not silently swallow failures

## Related

- `docs/platforms/backend/playbooks/add-update-fix.md`
- `docs/invariants/backend.md`
