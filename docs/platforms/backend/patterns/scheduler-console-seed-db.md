# Scheduler, Console, Seed, and DB Operations (Ejtmaa)

## 1) Scheduler

Config: `backend/src/resources/configs/scheduler/index.ts`, `backend/src/resources/configs/scheduler/cron.ts`

Current state:
- Example task `TestTask` registered in `cron.ts` for framework wiring
- `ExpireSubscriptionsTask` — every hour; sets `Subscription.status` from `ACTIVE` → `EXPIRED` when `ends_at <= now` (idempotent bulk update)
- Further production domain scheduler jobs land as modules are added

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

### `init()`

Seeds only:
- system settings defaults (about, terms, privacy, rights, seller_name = "Ejtmaa", VAT flags)
- default supervisor admin account when none exists

Does **not** seed demo customers or organizations.

### `update("test_seed")`

Targeted demo seed via `DatabaseConsole.update` → `SeedPatch.update("test_seed")`:

1. `seedDemoCustomers` — early-return if any customer exists
2. `seedDemoOrganizations` — early-return if any organization exists
3. `seedDemoMembers` — early-return if any member exists; explicit org `subdomain` calls + curated Saudi names; `organization.createMember(...)`
4. `seedDemoCatalogPlans` — early-return if any Plan exists; creates three catalog tiers via `Plan().create(...)` each with `monthly_price` + `yearly_price` (entry array named `demoCatalogPlanEntries`, not `plans`)

Demo row values stay in `SeedPatch.ts` only; docs do not mirror them.

Unknown `update` versions reject with `NOT_VALID`.

Seed logo directory: `backend/static/upload/__seed/images/` (upload gitignore whitelists `__seed/`).

Organization seed mechanism: `docs/platforms/backend/contracts/organization-domain.md` §7.  
Member seed mechanism: `docs/platforms/backend/contracts/member-domain.md` §6.  
Plan catalog seed mechanism: `docs/platforms/backend/contracts/plan-domain.md` §6.  
Subscription: no seed rows; expire task contract in `subscription-domain.md` §5.

## 4) Safety rules

- Jobs and seed actions must be idempotent (prefer early-return when target rows already exist)
- Use transactions for multi-step writes
- Observable logs for critical branches
- Do not silently swallow failures
- Keep demo data out of `init()` unless product bootstrap truly requires it
- Do not name seed structures `plan` / `plans` (conflicts with product ORM model `Plan`)
- Demo person names: curated realistic locale names in code — do not use faker for Saudi member/customer-facing demo people

## Related

- `docs/platforms/backend/playbooks/add-update-fix.md`
- `docs/platforms/backend/contracts/organization-domain.md`
- `docs/platforms/backend/contracts/member-domain.md`
- `docs/platforms/backend/contracts/plan-domain.md`
- `docs/platforms/backend/contracts/subscription-domain.md`
- `docs/invariants/backend.md`
