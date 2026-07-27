# `@nodejs/socket` Library Reference

## Purpose

Authoritative reference for the vendored realtime framework package
`backend/eng-hosam/@nodejs/socket` (version `5.0.0`, `socket.io` / `socket.io-client` `^4.8.0`).

It documents the full library surface (server path and client path), the runtime contracts a
controller or middleware author must satisfy, and the known limitations that already exist in the
checked-in code.

Ejtmaa domain wiring lives in `backend/src` and is summarized in §10; the socket event mirror
contract with the frontends stays in `docs/platforms/backend/contracts/socket-event-mirroring.md`.

## 1) Package layout and consumption

| Path | Role |
|---|---|
| `src/index.ts` | Public exports plus `getSocketServerDriver` / `getSocketClientDriver` |
| `src/SocketProvider.ts` | `ProviderBase` implementation registered under app alias `socket` |
| `src/types/Config.ts` | `Config`, `ServerDriverConfig`, `ClientDriverConfig`, `SocketListenerType`, `SocketEventListener` |
| `src/drivers/SocketServerDriverBase.ts` | Server driver base (`provider`, `config`, `boot()`) |
| `src/drivers/SocketClientDriverBase.ts` | Client driver base (`provider`, `config`, `boot()`) |
| `src/drivers/SocketIOServerDriver.ts` | Socket.IO server engine: namespaces, middlewares, routing, queueing |
| `src/drivers/SocketIOClientDriver.ts` | Socket.IO client engine: connect/disconnect routing, queueing |
| `src/controllers/SocketServerControllerBase.ts` | Server controller base |
| `src/controllers/SocketClientControllerBase.ts` | Client controller base |
| `src/middlewares/SocketServerMiddlewareBase.ts` | Bootable middleware base. Its `driver` field accepts either driver type, but only `SocketIOServerDriver` registers middlewares — `ClientDriverConfig` has no middleware surface |

Consumption facts:

- `backend/package.json` declares `"@nodejs/socket": "file:./eng-hosam/@nodejs/socket"`.
- `backend/node_modules/@nodejs/socket` is a **plain copy**, not a symlink or hardlink.
- Package `main` is `./lib/index.js`; `lib/` is **git-tracked** and produced by Babel
  (`build:types` + `build:js` in the package `package.json`).

Consequence: editing `src` alone changes nothing at runtime. See §12.

## 2) Provider composition

`SocketProvider` is registered in `backend/src/resources/configs/app.ts` under alias `socket`,
with config `backend/src/resources/configs/socket/index.ts`.

`Config` shape:

```ts
{
    serversDrivers: {
        [alias: string]: Importer
    },
    clientsDrivers: {
        [alias: string]: Importer
    },
    configs: {
        servers: {
            [alias: string]: Importer
        },
        clients: {
            [alias: string]: Importer
        }
    },
    logger?: "console" | string,
    env?: {
        logger?: string
    }
}
```

`boot()` runs `bootServers()` then `bootClients()`. For each alias it resolves the driver class and
its config through `ImporterHelper` (`@nodejs/core_app`), constructs
`new DriverClass(provider, config)`, and awaits `driver.boot()`.

`Importer` accepts a direct class reference, a `[value, refs]` tuple, or
`["dynamic", () => import(...), "default"]`.

Access points:

| Accessor | Meaning |
|---|---|
| `provider.socketServerDriver<D>(alias)` | Booted server driver by alias |
| `provider.socketClientDriver<D>(alias)` | Booted client driver by alias |
| `getSocketServerDriver<D>(alias)` | Same, resolved through `CURRENT_APP()` |
| `getSocketClientDriver<D>(alias)` | Same, resolved through `CURRENT_APP()` |

`provider.__Log()` resolves the logger provider channel from `config.env.logger` (env var name) or
`config.logger`, defaulting to `"console"`. Every swallowed driver error is reported through it at
`debug` level.

## 3) `ServerDriverConfig`

```ts
{
    port: any,
    https?: boolean,
    https_options?: () => Promise<HttpsServerOptions>,
    http_options?: () => Promise<HttpServerOptions>,
    options: () => Promise<{ [key: string]: any }>,
    env: {
        port?: string,
        https?: string
    },
    controllers: {
        [alias: string]: Importer
    },
    middlewares: {
        [alias: string]: (socket, next, refs) => any
    },
    bootableMiddlewares: {
        [alias: string]: Importer
    },
    namespaces: {
        [namespacePath: string]: {
            globalMiddlewares: Array<string | [string, any?]>,
            routes: {
                connection: string | [string, any?],
                [event: string]: string | [string, any?] | [string, any?, SocketListenerType?]
            }
        }
    },
    errorsFunnel?: (server, namespace, socket, next, err) => Promise<any>,
    configs?: {
        controllers?: { [alias: string]: Importer },
        bootableMiddlewares?: { [alias: string]: Importer }
    }
}
```

Resolution rules:

- `port()` returns `process.env[env.port] || config.port` when `env.port` is set, else `config.port`.
- `isHttps()` returns `process.env[env.https] === "true"` when `env.https` is set, else
  `config.https === true`.
- `options()` is awaited once and passed to `new Server(opts)`.
- Namespace keys are literal Socket.IO namespace paths (`/customer`), used directly by `io.of(key)`.
- `routes.connection` is mandatory by type; the driver always resolves it in `routes()`.

## 4) Server boot lifecycle

`SocketIOServerDriver.boot()` runs in fixed order:

1. `options()` is awaited, then `new Server(opts)` is constructed.
2. `bootControllers()` — one **singleton** controller instance per alias, each `boot()`ed.
3. `bootMiddlewares()` — see §5.
4. `bootNamespaces()` — for every namespace: `io.of(path)`, register in `onlineNamespaces`,
   initialize `socketsMap[path] = {}`, apply global middlewares, then bind the `connection` route.
5. `runIOServer()` — the driver creates its **own** `http`/`https` server, calls `io.listen(server)`
   and `server.listen(port())`. The promise resolves on `listen`, and the bound URL is logged.

HTTPS mode reads `https_options()`; HTTP mode reads `http_options()` (both optional, default `{}`).

## 5) Middleware contract

Two registries feed one runtime map, `onlineMiddlewares`:

| Config key | Shape | Behavior |
|---|---|---|
| `middlewares` | `(socket, next, refs) => any` | Stored verbatim as `_handle`, no boot, no error funnel wrapper |
| `bootableMiddlewares` | `Importer` of a `SocketServerMiddlewareBase` subclass | Constructed, `boot()`ed, wrapped in a `_handle` that instantiates per connection |

Because both write to the same map and bootable middlewares are registered second, a duplicated
alias silently resolves to the bootable one.

Bootable middleware execution per handshake:

```ts
middleware
    .newInstance()
    .setEventData(socket, next, refs)
    .handle()
```

Rules for a `SocketServerMiddlewareBase` subclass:

- `handle()` **must** call `this.next()` on success; the driver never calls it for you.
- A throw (or rejected promise) goes to `errorsFunnel(io, undefined, socket, next, err)` when
  configured, otherwise the driver calls `next(err)` itself. When a funnel is configured, the funnel
  owns the decision to reject the handshake.
- `this.refs` carries the second tuple element from `globalMiddlewares: [alias, refs]`.
- `this.shared` is the boot-time object shared by reference with every per-connection instance.

`globalMiddlewares` are applied through `namespace.use(...)` in declaration order. An unknown alias
rejects the connection with `Socket middleware "<alias>" was not found`.

## 6) Route grammar and resolution

Accepted route values (server `routes` and client `routes` alike):

```ts
event: "controllerAlias"
event: ["controllerAlias", refs]
event: ["controllerAlias", refs, "on" | "once"]
event: "controllerAlias@customHandler"
```

`getRouteData(namespaceAlias, eventAlias)` resolves:

| Output | Source |
|---|---|
| `controller` | `onlineControllers[alias]` — throws when missing |
| `refs` | tuple index `1`, default `{}` |
| `type` | tuple index `2`, default `"on"`; anything other than `on` / `once` throws |
| `handle` | method name after `@`, default `"handle"`; malformed `a@` or `a@b@c` throws |

The `@method` form lets one controller class serve several events with different entry methods.

## 7) The handler array — declarative listener state machine

The value returned by a controller handler is **not** a result payload. It is the complete set of
event aliases that must be bound to that socket from now on.

```ts
async handle() {
    // ...
    return [
        "meeting.live.sync",
        "meeting.live.update"
    ];
}
```

`syncChildRoutes(namespaceAlias, namespace, events, socket)` diffs the returned set against the
listeners currently registered for that socket:

1. every registered alias **absent** from the returned array is `socket.off(...)`-ed and dropped;
2. every returned alias **not** yet registered is resolved via `getRouteData` and bound with
   `socket.on` (or `socket.once` when `type === "once"`).

Direct consequences:

- Returning `[]` or `undefined` removes **all** event listeners; the socket stays connected but inert.
- The array is absolute, not additive: an event that must survive has to be returned by every
  handler that runs while it should stay bound.
- Returning an alias that is not declared in `namespaces[ns].routes` throws
  `Socket route "<alias>" was not found in namespace "<ns>"`; the throw is caught by the surrounding
  error path, so the listener simply never binds.
- Per-socket listener registries live in a `WeakMap<Socket, Map<string, listener>>` and are cleared
  on `disconnect` together with the execution queue.

Connection flow, per accepted socket:

```ts
controller
    .newInstance()
    .setEventData(io, namespace, socket, "connection", {}, refs)
    [handle]()
    .then(events => syncChildRoutes(ns, namespace, events ?? [], socket))
    .catch(err => errorsFunnel?.(io, namespace, socket, undefined, err))
```

Child event flow adds two guards: `once` listeners are removed from the registry before the handler
runs, and the post-handler sync is skipped when `socket.connected` is already `false`.

## 8) Execution model and controller state

- **Serialized per socket.** `enqueueSocketExecution` chains every event of a given socket onto one
  promise queue, so two events from the same client never interleave. A failed task does not break
  the chain (`.catch` keeps the queue alive), and the queue entry is deleted once it drains.
- **Instance per event.** `controller.newInstance()` creates a fresh controller for every handled
  event; only `shared` is carried over by reference from the booted singleton.
- **Event payload normalization.** `args.length <= 1 ? args[0] : args`. A single emitted argument
  arrives as itself, multiple arguments arrive as an array.

`SocketServerControllerBase` fields after `setEventData(server, namespace, socket, event, args, refs)`:

| Field | Value |
|---|---|
| `server` | the `socket.io` `Server` |
| `namespace` | the `Namespace` instance |
| `socket` | the connected `Socket` |
| `event` | the resolved event alias (`"connection"` on connect) |
| `args` | normalized payload |
| `refs` | route tuple refs |
| `driver` / `config` / `shared` | boot-time values |

Project controllers narrow the loose base types with `declare`, for example
`backend/src/app/socket/controllers/ConnectionIOController.ts`:

```ts
declare socket: Socket<DefaultEventsMap, DefaultEventsMap, DefaultEventsMap, SocketData>;
declare driver: SocketIOServerDriver;
declare namespace: Namespace;
```

## 9) Error handling

`errorsFunnel(server, namespace, socket, next, err)` receives:

| Origin | `namespace` | `next` |
|---|---|---|
| Bootable middleware | `undefined` | the handshake `next` |
| Connection controller | the `Namespace` | `undefined` |
| Child event controller | the `Namespace` | `undefined` |

Every path also logs through `provider.__Log().debug(err)`. There is no built-in client-facing error
emit: whatever the funnel does not send, the client never learns.

The Ejtmaa funnel in `backend/src/resources/configs/socket/io.ts` is `next?.(err)`, so middleware
failures reject the handshake while connection and event failures are log-only.

## 10) Ejtmaa server wiring

Config: `backend/src/resources/configs/socket/io.ts` (server driver alias `io`, port `6000` default,
env `IO_PORT` / `INSTALL_IO_PORT`, `HTTPS` toggle, `transports: ["websocket"]`).

| Namespace | Global middlewares | `connection` controller | Room joined | Child events (bound by connection return) |
|---|---|---|---|---|
| `/customer` | `auth` | `ConnectionIOController` | `Rooms.CUSTOMER(customerId)` | none (`return []`) |
| `/supervisor` | `auth` | `ConnectionIOController` | `Rooms.ALL_SUPERVISORS` | none (`return []`) |
| `/meeting` | `meeting_auth` | `MeetingConnectionIOController` | `Rooms.MEETING(meetingId)` | `meeting.live.sync` → `meeting_live_sync`, `meeting.live.update` → `meeting_live_update` |

`/customer` and `/supervisor` connection controllers still return `[]`.

`/meeting` connection returns both live aliases, which binds the child routes declared in the same namespace.

Every `/meeting` controller extends `backend/src/app/socket/controllers/meeting/MeetingIOControllerBase.ts`, which holds the bound-event tuple in one place and exposes `meetingBoundEvents({ only?, except? })`. Because the handler-array return is absolute (§7), each handler returns that same set — including the rejection path `rejectLive(code)`, which emits `meeting.live.error` and still returns the full set so a refused write never leaves the socket inert. Contract: `docs/platforms/backend/contracts/meeting-realtime-socket.md` §3.

Meeting domain controllers live under `backend/src/app/socket/controllers/meeting/`. Register every new meeting event in `io.ts` (`controllers` + `/meeting` `routes`) with a dotted event name (`meeting.*`) and add it to the base tuple so it stays bound.

`AuthenticationIOMiddleware` (`backend/src/app/socket/middlewares/AuthenticationIOMiddleware.ts`)
is token-only for actor namespaces: a `token` header/query resolves the customer or supervisor actor and sets
`isAuthed`. Missing/invalid token throws `NOT_VALID_CREDENTIAL`. Actor `SocketData` has no organization field.

`MeetingAuthenticationIOMiddleware` (`meeting_auth`) owns `/meeting` handshake proof (member token + meeting + roster + ACTIVE org) and exports `MeetingSocketData` plus `currentOrganization` / `currentMeeting` / `currentMember` / `currentParticipant`.

Namespace, FCM namespace, and room name constants are centralized in
`backend/src/resources/consts/NotificationsConsts.ts` (`Namespaces.MEETING`, `FCM_Namespaces.MEETING`, `Rooms.MEETING`).

### Outbound emit path

The library exposes no emit helper. Broadcasting goes through `@nodejs/notify`
(`backend/eng-hosam/@nodejs/notify/src/broadcasters/SocketIOBroadcaster.ts`), which pulls the `io`
server driver from the socket provider at boot and emits with
`driver.onlineNamespaces[namespace].to(room).emit(event, data)`. Event classes under
`backend/src/app/notify/events/` declare `channels()` as `{ namespace, room }` pairs built from
`NotificationsConsts`.

## 11) Client driver (available, unused in Ejtmaa)

`SocketIOClientDriver` is fully implemented but not wired: `clientsDrivers` and `configs.clients`
are empty in `backend/src/resources/configs/socket/index.ts`. Documented here as an available
surface for future outbound socket clients.

`ClientDriverConfig` requires `host`, `namespace`, `options()`, `env` (`host` / `namespace` env var
names), `controllers`, and `routes` containing **both** `connect` and `disconnect`.

Behavioral differences from the server driver:

| Aspect | Client driver |
|---|---|
| Target URL | `host()` = env-or-config host with trailing slashes stripped, plus normalized `namespace()`; a `/` namespace leaves the host bare |
| Controller base | `SocketClientControllerBase` — `setEventData(namespaceAlias, socket, event, args, refs)` passes the namespace **string**, and has no `server` field |
| Boot order | `bootControllers()` runs **before** the socket is created, then `routes()` binds `connect` / `disconnect` |
| Listener state | One `Map` for the single socket, not a per-socket `WeakMap` |
| Queue | One driver-wide promise queue, reset to `Promise.resolve()` on disconnect |
| Reconnect safety | `connectionGeneration` increments on every `connect` and `disconnect`; a handler that finishes after a reconnect has its `syncChildRoutes` skipped |
| Disconnect | Clears all child listeners first, then runs the `disconnect` controller with `{ reason, details }`; its return value is ignored |
| Errors | `errorsFunnel(namespaceAlias, socket, err)` — three arguments, no `next` |

The `connect` handler owns the same handler-array contract described in §7.

## 12) Change workflow for this package

1. Edit `backend/eng-hosam/@nodejs/socket/src`.
2. Type-check inside the package: `yarn type-check` (script exists in the package).
3. Build: `yarn build` (`build:types` + `build:js`) — `lib/` is tracked and must be committed with
   the `src` change, or the runtime keeps the old behavior.
4. Refresh the copy in `backend/node_modules/@nodejs/socket` (yarn install for the `file:`
   dependency); the copy is not linked, so a package build alone does not reach the app.
5. Type-check the consumer: `yarn type-check` in `backend/`.

Do not hand-edit `lib/` — it is Babel output.

## 13) Known limitations

| # | Limitation | Evidence |
|---|---|---|
| 1 | No Socket.IO acknowledgement support. An ack callback sent by the client lands inside `args` and is never invoked by the framework. | `args.length <= 1 ? args[0] : args` in both drivers |
| 2 | `driver.socketsMap[namespace]` is created per namespace but never populated by the library. | `bootNamespaces()` only assigns `{}`; the only writer is a commented-out line in `ConnectionIOController` |
| 3 | Consequently, the id-targeted branch of `SocketIOBroadcaster.handle()` (taken when a `Channel` carries `id`) stays unreachable, and it is unused anyway — Ejtmaa events declare `{ namespace, room }` only. Room emit is the only working outbound path. The lookup defect behind it is fixed: `@nodejs/notify` `7.0.0` reads `namespace.sockets.get(socketId)` instead of the `Namespace.connected` property removed in socket.io v3, so enabling the path now needs only a `socketsMap` writer on connect plus a delete on disconnect. Per-actor delivery is already covered by dedicated rooms (`Rooms.CUSTOMER(id)`), which also handle multi-device clients. | `@nodejs/notify` `7.0.0` broadcaster, `Channel` type |
| 4 | Connection and child-event failures are invisible to the client unless `errorsFunnel` emits something; Ejtmaa's funnel cannot (its `next` is `undefined` on those paths). | §9 |
| 5 | A duplicated alias across `middlewares` and `bootableMiddlewares` resolves to the bootable one with no warning. | `bootMiddlewares()` registration order |
| 6 | No dynamic namespaces (`io.of(regex)`) and no room helpers: namespaces are static config keys and rooms are joined manually inside controllers. | `bootNamespaces()` iterates `Object.keys(config.namespaces)` |

## 14) Traceability map

| Path | Where documented |
|---|---|
| `src/index.ts` | §1, §2 (accessors) |
| `src/SocketProvider.ts` | §2 |
| `src/types/Config.ts` | §2, §3, §11 |
| `src/drivers/SocketServerDriverBase.ts` | §1 (base surface only: `provider`, `config`, empty `boot()`) |
| `src/drivers/SocketClientDriverBase.ts` | §1 (same, client side) |
| `src/drivers/SocketIOServerDriver.ts` | §3–§9 |
| `src/drivers/SocketIOClientDriver.ts` | §11 |
| `src/controllers/SocketServerControllerBase.ts` | §7, §8 |
| `src/controllers/SocketClientControllerBase.ts` | §11 |
| `src/middlewares/SocketServerMiddlewareBase.ts` | §5 |
| `lib/**.js` / `lib/**.d.ts` | §1, §12 — Babel and `tsc` build output, tracked, never hand-edited |
| `lib/tsconfig.tsbuildinfo` | Incremental build cache. Not behavioral; regenerated by `build:types` |
| `package.json` | §1, §12 |
| `.gitignore` | Package-local ignores (`node_modules`, IDE files, scratch `t.ts`, self-signed cert pair). Not behavioral |
| `backend/src/resources/configs/socket/index.ts` | §2, §11 |
| `backend/src/resources/configs/socket/io.ts` | §3, §9, §10 |
| `backend/src/app/socket/controllers/ConnectionIOController.ts` | §8, §10 |
| `backend/src/app/socket/controllers/meeting/MeetingIOControllerBase.ts` | §10 |
| `backend/src/app/socket/controllers/meeting/MeetingConnectionIOController.ts` | §10 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveSyncIOController.ts` | §10 |
| `backend/src/app/socket/controllers/meeting/MeetingLiveUpdateIOController.ts` | §10 |
| `backend/src/app/socket/middlewares/AuthenticationIOMiddleware.ts` | §10 |
| `backend/src/app/socket/middlewares/MeetingAuthenticationIOMiddleware.ts` | §10 |
| `backend/src/resources/consts/NotificationsConsts.ts` | §10 |

## Related

- `docs/platforms/backend/modules/runtime-integrations.md` — provider-level runtime summary
- `docs/platforms/backend/contracts/socket-event-mirroring.md` — backend → frontend event mirror
- `docs/platforms/backend/contracts/meeting-realtime-socket.md` — `/meeting` namespace contract
- `docs/platforms/website/organization-host-routing.md` — website `/meeting` consumer + org-host boot contract
- `.cursor/rules/nodejs-socket-handler-contract.mdc`
- `.cursor/rules/nodejs-socket-namespace-registration.mdc`
- `.cursor/rules/eng-hosam-vendored-package-sync.mdc`
- `.cursor/skills/nodejs-socket-server-event/SKILL.md`
- `.cursor/skills/nodejs-socket-client-driver/SKILL.md`
- `.cursor/skills/meeting-realtime-socket/SKILL.md`
- `.cursor/rules/meeting-realtime-socket.mdc`
