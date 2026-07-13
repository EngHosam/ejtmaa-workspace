---
name: socket-event-mirroring
description: Keeps backend user-facing socket events mirrored into website and cpanel contracts. Use when adding, removing, renaming, or reviewing OnCustomerEvent, OnUserEvent, backend notify events, socket payload types, or frontend socket event registries.
---

# Socket Event Mirroring

## When to Use

- Adding, renaming, or removing a backend notify/socket event.
- Updating frontend socket payload unions in `website/` or `cpanel/`.
- Reviewing drift between `backend/src/app/types/Events.ts` and client mirrors.

## Read first

- `docs/platforms/backend/contracts/socket-event-mirroring.md`
- `.cursor/rules/socket-event-mirroring.mdc`
- `backend/src/app/types/Events.ts`
- `backend/src/app/notify/events/**/*.ts`

## Instructions

1. Identify the backend source-of-truth change: event name and payload shape.
2. Mirror by audience:
   - customer -> `website/src/types/events.ts` and `website/src/resources/configs/socket/events.ts`
   - supervisor -> `cpanel/src/types/events.ts` and `cpanel/src/resources/configs/socket/events.ts`
3. Remove stale placeholders from touched mirror registries.
4. Update `docs/platforms/backend/contracts/socket-event-mirroring.md` and any inventory/runtime docs that cite the mirror paths.
5. Verify with existing scripts only: `yarn type-check` in each touched platform repo.

## Non-negotiable rules

- Do not invent frontend-only event names.
- Do not mirror supervisor events into `website/`.
- Customer events mirror into `website/` only; do not mirror customer events into `cpanel/`.
- Payload unions must match backend exactly.
