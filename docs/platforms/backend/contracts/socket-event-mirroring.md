# Socket Event Mirroring Contract

## Purpose

Defines how backend user-facing socket events are mirrored into `website/` (customer) and `cpanel/` (supervisor).

## Backend source of truth

- `backend/src/app/types/Events.ts`
- `backend/src/app/notify/events/**/*.ts`

Frontend mirrors must match backend event names and payload unions exactly.

## Mirror targets

### Customer (`website/`)

- `website/src/types/events.ts`
- `website/src/resources/configs/socket/events.ts`

Backend event: `OnCustomerEvent`

### Supervisor (`cpanel/`)

- `cpanel/src/types/events.ts`
- `cpanel/src/resources/configs/socket/events.ts`

Backend event: `OnUserEvent`

## Workflow

1. Update backend event truth first.
2. Mirror into the correct frontend by audience.
3. Update `types.ts` and matching `events.ts` registry together.
4. Update this contract in the same task.
5. Run existing `type-check` scripts for touched packages.

## Event payloads (from `Events.ts`)

### `OnCustomerEvent`

Payload union:

- `UPDATED`

### `OnUserEvent`

Payload union:

- `NEW_CUSTOMER`

Supervisor cpanel mirrors consume `OnUserEvent` for supervisor broadcast notifications.

## Anti-drift rules

- Event names and payload unions must match backend exactly.
- Do not leave placeholder events when real events exist.
- Do not update only `types.ts` without the matching `events.ts` registry.

## Related

- `.cursor/rules/socket-event-mirroring.mdc`
- `.cursor/skills/socket-event-mirroring/SKILL.md`
- `docs/platforms/backend/modules/runtime-integrations.md`
