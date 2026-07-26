# Socket Event Mirroring Contract

## Scope boundary

This contract covers **backend → frontend outbound** user-facing notify/socket events (`OnCustomerEvent` / `OnUserEvent`).

**Out of scope:** client → server inbound emits such as `/meeting` `meeting.join`. Those are registered only in `backend/src/resources/configs/socket/io.ts` and consumed by server controllers; do not add them to frontend event registries. See `docs/platforms/backend/contracts/meeting-realtime-socket.md`.

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
