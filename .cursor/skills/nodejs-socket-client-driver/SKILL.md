---
name: nodejs-socket-client-driver
description: >-
  Wires an outbound Socket.IO client on @nodejs/socket SocketIOClientDriver — clientsDrivers
  registration, ClientDriverConfig host/namespace/env resolution, connect and disconnect
  controllers, and the reconnect generation guard. Use when the backend must consume a remote
  socket server, since clientsDrivers is currently empty in Ejtmaa.
---

# `@nodejs/socket` client driver

## When to Use

- The backend must connect **out** to a remote Socket.IO server as a client.
- Reviewing or debugging an existing client driver registration.

Ejtmaa currently registers no client driver (`clientsDrivers: {}` in
`backend/src/resources/configs/socket/index.ts`). Treat every step here as new wiring.

## Read first

- `docs/platforms/backend/modules/nodejs-socket-library.md` §11 — client driver contract
- `.cursor/rules/nodejs-socket-handler-contract.mdc` — the handler-array contract applies here too
- `backend/eng-hosam/@nodejs/socket/src/drivers/SocketIOClientDriver.ts`

## Instructions

1. **Register the driver** in `backend/src/resources/configs/socket/index.ts`:
   `clientsDrivers: { <alias>: SocketIOClientDriver }` plus a matching entry in
   `configs.clients.<alias>`.

2. **Write the client config** as a `ClientDriverConfig`:
   - `host`, `namespace`, and `env` (`host` / `namespace` env var **names**) — the driver strips
     trailing slashes from the host, normalizes the namespace, and concatenates them
   - `options()` for the `socket.io-client` options
   - `controllers` aliases
   - `routes` containing **both** `connect` and `disconnect`
   - optional `errorsFunnel(namespaceAlias, socket, err)` — three arguments, no `next`

3. **Write controllers** extending `SocketClientControllerBase<any>`. The signature differs from the
   server side: `setEventData(namespaceAlias, socket, event, args, refs)` gives a namespace
   **string** and no `server` field.

4. **`connect` owns the listener set.** Its returned array is the complete set of event aliases to
   bind, diffed exactly like the server path. The `disconnect` handler's return value is ignored,
   and its `args` are `{ reason, details }`.

5. **Respect the generation guard.** Every connect and disconnect increments
   `connectionGeneration`; a handler that resolves after a reconnect has its listener sync skipped.
   Never cache the socket or rebind listeners manually outside the returned array.

6. **Keep secrets in env.** Remote host and credentials come from environment variables named in
   `env`, never hardcoded, and must be added to `backend/.env.example`.

7. **Document and verify.** Add the new client surface to
   `docs/platforms/backend/modules/nodejs-socket-library.md` §11 and the env keys to
   `docs/platforms/backend/modules/runtime-integrations.md` §2, then run `yarn type-check` in
   `backend/`.

## Non-negotiable rules

- One driver-wide execution queue serializes all events; do not assume concurrency.
- Do not reuse a server controller class for a client route — the `setEventData` contracts differ.
- Do not rely on acknowledgements; the client driver has the same no-ack limitation.
