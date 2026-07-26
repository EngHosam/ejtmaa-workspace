---
name: nodejs-socket-server-event
description: >-
  Adds or changes an Ejtmaa realtime server surface on @nodejs/socket — namespace,
  bootable middleware, connection controller, child event controller, room join, and
  the handler-array listener set. Use when wiring socket routes in
  backend/src/resources/configs/socket/io.ts, writing controllers under
  backend/src/app/socket, debugging a listener that never fires, or reviewing socket
  registration drift.
---

# `@nodejs/socket` server event

## When to Use

- Adding a new namespace, event, or connection controller to the backend socket driver.
- Adding or changing a bootable socket middleware (handshake auth, tenant resolution).
- Debugging a socket event that never fires, fires twice, or stops firing after another event.
- Reviewing whether socket registration, rooms, and emit paths still match the contract.

## Read first

- `docs/platforms/backend/modules/nodejs-socket-library.md` — full framework reference
- `.cursor/rules/nodejs-socket-handler-contract.mdc`
- `.cursor/rules/nodejs-socket-namespace-registration.mdc`
- `backend/src/resources/configs/socket/io.ts` — the only registration point
- `backend/src/app/socket/controllers/OrgConnectionIOController.ts` — reference org connection controller
- `backend/src/app/socket/controllers/meeting/MeetingJoinIOController.ts` — reference meeting child event

## Instructions

1. **Decide the surface.** New namespace, new event on an existing namespace, or a change to an
   existing connection controller. A namespace key is the literal Socket.IO path.

2. **Write the controller** under `backend/src/app/socket/controllers/` (domain subfolders allowed, e.g. `controllers/meeting/`), extending
   `SocketServerControllerBase<any>`. Narrow the loose base fields with `declare` (`socket` typed
   with the `SocketData` payload from `AuthenticationIOMiddleware`, `driver`, `namespace`).
   Read the request from `this.args`, `this.socket.data`, and `this.refs`. Prefer dotted event
   names for domain groups (`meeting.join`, `meeting.leave`).

3. **Return the listener set.** `handle()` returns the complete array of event aliases that must
   stay bound to this socket — `[]` unbinds everything. Every returned alias must be declared in the
   same namespace `routes`. Never return a domain payload.

4. **Register in `io.ts`** in one place:
   - controller alias under `controllers`
   - namespace entry with `globalMiddlewares` (include `auth` when actor or tenant data is needed)
     and `routes.connection`
   - child events as `event: "alias"`, `["alias", refs]`, `["alias", refs, "once"]`, or
     `"alias@method"`

5. **Join rooms from constants.** Use `Rooms` / `Namespaces` in
   `backend/src/resources/consts/NotificationsConsts.ts`; add a constant instead of inlining a room
   string. Throw (for example `NOT_VALID_CREDENTIAL`) when the expected `socket.data` actor or
   organization is missing.

6. **Outbound messages** go through a `@nodejs/notify` event class under
   `backend/src/app/notify/events/` with `channels()` returning `{ namespace, room }`. Room emit is
   the only working path; do not use id-targeted emit (see the reference, §13).

7. **Mirror user-facing events** into the frontends per
   `.cursor/skills/socket-event-mirroring/SKILL.md`.

8. **Update docs** in the same task: the namespace table in
   `docs/platforms/backend/modules/nodejs-socket-library.md` §10 and, when the provider summary
   changes, `docs/platforms/backend/modules/runtime-integrations.md` §5.

9. **Verify** with existing scripts only: `yarn type-check` in `backend/`, plus the frontend
   packages touched by a mirror change.

## Debug checklist

| Symptom | First check |
|---|---|
| Event never fires | Is the alias returned by the handler that ran last, and declared in `routes`? |
| Listener disappeared | A later handler returned an array without that alias — the set is absolute |
| Fires once only | Route registered with listener type `once` |
| Handler throws, client sees nothing | Connection/event errors are log-only; emit an explicit failure event |
| Two events interleave badly | They cannot — one serialized queue per socket; look for shared state instead |
| State lost between events | A fresh controller instance runs per event; only `shared` and `socket.data` survive |
| Ack callback never called | Acks are unsupported by the framework |

## Non-negotiable rules

- Do not register a controller anywhere but `backend/src/resources/configs/socket/io.ts`.
- Do not reuse one alias across `middlewares` and `bootableMiddlewares`.
- Do not store per-socket state in the controller's `shared`.
- Do not modify `backend/eng-hosam/@nodejs/socket` to solve an app-level problem; if it is truly
  unavoidable, follow `.cursor/rules/eng-hosam-vendored-package-sync.mdc`.
