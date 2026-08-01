# Website Flow — Meeting Broadcast (LiveKit client)

Organization-host live A/V surface inside the Meeting shell. Shell / session context: `organization-host-routing.md` §5.3–§5.5. Join-token backend contract: `../backend/contracts/livekit-media-plane.md` §6. Media-plane boundaries (LiveKit is A/V only): same page §2.

## 1) Scope

**Shipped**

- Website dependency `livekit-client@2.21.0` (exact pin) + lockfile entries.
- Backend join-token response now carries the client connect URL: `{ token, url }`.
- `useMeetingLiveKitToken` is a discriminated union — `ready` guarantees both `token` and `url`; every other status has both `null`.
- New central room hook `useMeetingLiveKitRoom` — one `Room` per ready `(token, url)`, connection status, local/remote peer projection, camera/mic publish toggles, local sound (playback) toggle, cooperative mute-all over the LiveKit data channel.
- `MeetingLiveBroadcast` replaced its temporary token probe with the real stage: featured chair tile + remote grid, per-track `<video>` / `<audio>` attach components, avatar placeholders, three control switches (camera / mic / sound), chair mute-all group.
- Session capability `can.muteAllMedia` (chair + `STARTED`) gates the mute-all group.
- `LaneFailed` accepts optional `title` / `description`; broadcast failure and `MeetingLinkingScreen` FAILED both use it instead of hand-rolled icon + message chrome.
- i18n `ui.layouts.meetingLayout.broadcast.*` (ar + en) and `meetingLayout.linking.failedHint`.

**Not shipped**

- Server-side or peer-side authority for mute-all (see §10.1).
- LiveKit `roomAdmin` moderation (`mutePublishedTrack`, `removeParticipant`) from the website.
- Screen share, active-speaker detection, device pickers, volume sliders, per-peer mute, pin/spotlight, recording / egress.
- Publish grants derived from `MeetingParticipant.type` (every roster member still gets LiveKit's default publish + subscribe grant).
- Media-driven attendance writes (`attended_at` / `left_at` stay on the session/socket path).
- In-UI retry for a failed room connection (page reload only), and any LiveKit `Room` tuning (`webAudioMix`, `adaptiveStream`, `dynacast` all stay at library defaults).

## 2) Entry points

| Layer | Path / symbol |
|---|---|
| Mount owner | `website/src/app/ui/pages/Meeting.tsx` (`MeetingContent`) — mounts `MeetingLiveBroadcast` while `meeting.status === "STARTED"` |
| Stage component | `website/src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` |
| Room hook | `website/src/app/ui/components/meeting/hooks/useMeetingLiveKitRoom.ts` |
| Token hook | `website/src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts` |
| Session (capability + roster) | `website/src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` |
| Org chrome | `website/src/app/ui/components/meeting/hooks/useOrganization.ts` |
| Failure chrome | `website/src/app/ui/components/Wrong.tsx` (`LaneFailed`) |
| Loading chrome | `website/src/app/ui/components/Loadable.tsx` |
| HTTP surface | `POST /website/custom/org/livekit_token` (`API.CUSTOM.ORG_LIVEKIT_TOKEN`) |

Mount chain: `MeetingLayout` (`MeetingLiveProvider` → `MeetingLiveSessionProvider` → `MeetingPageProvider`) → `MeetingShell` READY tree → `Meeting.tsx` `STARTED` stack → `MeetingLiveBroadcast`. The broadcast never mounts before linking is READY and never mounts outside `STARTED`.

## 3) Dependency

| Package | Version | Role |
|---|---|---|
| `livekit-client` | `2.21.0` (exact) | Browser SDK: `Room`, `RoomEvent`, `ConnectionState`, `Participant`, `Track` |

Transitive additions in `website/yarn.lock`: `@livekit/protocol@1.50.4`, `@livekit/mutex@1.1.1`, `@bufbuild/protobuf@^1.10.0`, `jose@^6.1.0`, and an `events@^3.3.0` range folded into the existing `events@3.3.0` entry.

`livekit-server-sdk` stays backend-only — the website never mints a token. `MeetingLiveBroadcast` imports `Track` as `import type` (it renders tracks, it does not construct a room); only the hook imports runtime symbols.

## 4) Token → connect contract

Backend `MeetingLiveKitTokenController` returns `{ token, url }`, where `url` is `LiveKitHelper.clientUrl()` (normalized to `ws` / `wss`). Consequence: the website has **no** LiveKit URL env key and must not hardcode a host — the connect URL always arrives with the JWT that authorizes it.

`useMeetingLiveKitToken` public shape:

```ts
| { status: "idle" | "pending" | "error"; token: null; url: null }
| { status: "ready";                     token: string; url: string }
```

| `status` | When |
|---|---|
| `idle` | `!can.enterLive`, or session `meeting.status !== "STARTED"`, or a missing route id (`memberId` / `memberToken` / `meetingId`) — **no** HTTP request |
| `pending` | Request in flight, or quiet network retry (3s) while still active |
| `ready` | Axios `READY` and **both** `token` and `url` mapped |
| `error` | Axios `ERROR` with a response body (`isResType`) — business/auth failure, no retry |

The union is the contract that keeps `Room.connect(url, token)` from being called with a partial payload: a response missing `url` degrades to `pending` (retry-eligible), never to a broken connect. Quiet retry uses `skipNetworkToast` (declared in `website/src/types/extends/global.ts`, honored in `resources/configs/axios.ts`).

## 5) Room hook — `useMeetingLiveKitRoom`

Takes **no arguments**. All inputs come from the token hook (which reads the live session). Single source of LiveKit state for the broadcast; the component holds no room, track, or connection state of its own.

### 5.1 Public shape

| Field | Type | Meaning |
|---|---|---|
| `status` | `"idle" \| "connecting" \| "connected" \| "error"` | Stage state (derived — see §5.2) |
| `videoStatus` | `"enabled" \| "muted" \| "unauthorized"` | Local camera **publish** state |
| `micStatus` | same | Local microphone **publish** state |
| `soundStatus` | `"enabled" \| "muted"` | Local **playback** of remote audio (never a publish state) |
| `local` | `MeetingLiveKitPeer \| null` | Local participant projection |
| `remotes` | `Record<string, MeetingLiveKitPeer>` | Remote peers keyed by LiveKit identity (= member id string) |
| `changeVideoStatus` / `changeMicStatus` | `(status: MeetingLiveKitToggleStatus) => void` | Publish toggles |
| `changeSoundStatus` | same signature | Local playback toggle |
| `muteAllVideos` / `muteAllMics` | `() => void` | Publish a cooperative mute-all command |

`MeetingLiveKitToggleStatus = Exclude<MeetingLiveKitMediaStatus, "unauthorized">` — callers may ask for `enabled` / `muted` only; `unauthorized` is an outcome the hook reports, never an input.

`MeetingLiveKitPeer` = `{ participantId, name, videoTrack, micTrack }`.

### 5.2 Status derivation

Internal `status` follows `RoomEvent.ConnectionStateChanged`:

| `ConnectionState` | Internal status |
|---|---|
| `Connecting`, `Reconnecting`, `SignalReconnecting` | `connecting` |
| `Connected` | `connected` (+ peer sync) |
| `Disconnected` | `disconnectRoom("error")` |

The returned value is a projection, not the raw state:

```ts
const visibleStatus = ready ? status : tokenFailed ? "error" : "idle";
```

Two reasons, both load-bearing: React effects settle a frame after the token drops (a stale `connected` must not leak into the UI), and a rejected token never reaches the room at all, so token `error` has to surface as a broadcast error instead of an eternal spinner.

`error` is terminal for the mounted stage — there is no reconnect entry point (§10.3).

### 5.3 Peer projection (`readPeer`)

Read from publications, not from cached track events:

| Field | Source | Rule |
|---|---|---|
| `participantId` | `participant.identity` | LiveKit identity is the member id string minted by the backend |
| `name` | `participant.name \|\| participant.identity` | Display name resolution prefers the session roster in the component (§6.3) |
| `videoTrack` | `getTrackPublication(Track.Source.Camera)` | **`null` when the publication is muted** — a muted camera produces no frames, so the tile must fall back to the avatar |
| `micTrack` | `getTrackPublication(Track.Source.Microphone)` | Kept even while muted — dropping it would detach the element and tear down audio playback (§6.2) |

`syncPeers` rewrites both `local` and `remotes` on every relevant room event: `ParticipantConnected`, `ParticipantDisconnected`, `TrackSubscribed`, `TrackUnsubscribed`, `TrackMuted`, `TrackUnmuted`, `LocalTrackPublished`, `LocalTrackUnpublished`, and on `Connected`.

### 5.4 Publish control

`setPublishEnabled(kind, enable)` requires a room in `ConnectionState.Connected` (`connectedRoom` guard), then calls `setCameraEnabled` / `setMicrophoneEnabled`:

- resolve → status becomes `enabled` / `muted` and peers re-sync,
- reject (denied browser permission, missing device) → status becomes `unauthorized`.

Nothing auto-publishes: the room connects with no camera and no mic, so a participant is silent and dark until they press a control. Both statuses start at `muted`.

### 5.5 Mic vs sound (two different controls)

| Control | State | Effect |
|---|---|---|
| Mic | `micStatus` | Publishes / mutes **my** microphone for everyone else |
| Sound | `soundStatus` | Mutes **my** playback of remote audio only — never touches any publication |

`soundStatus` starts at `muted` on mount and on every `resetMedia`, deliberately: browsers block autoplay of audio, so the first user gesture must be the one that starts playback. `changeSoundStatus("enabled")` sets state, then calls `room.startAudio()` **inside the click stack**; if the browser refuses, the promise rejects and the status reverts to `muted` so the button never lies about being on.

### 5.6 Mute-all (cooperative, data channel)

| Command | Payload string |
|---|---|
| Cameras | `ejtmaa.livekit.mute_all_videos` |
| Microphones | `ejtmaa.livekit.mute_all_mics` |

Sent with `localParticipant.publishData(..., { reliable: true })`; LiveKit delivers to other participants only, so the sender's own media is untouched. A receiver decodes the payload and applies `setPublishEnabled(kind, false)` locally. Unknown payloads are ignored.

This is a request, not enforcement — see the ceilings in §10.1 and §10.2.

### 5.7 Lifecycle

One effect owns the room, keyed on `ready` / `token` / `url` plus the stable callbacks:

1. Not ready → `disconnectRoom("idle")` and stop.
2. `new Room()` (library defaults), `status = "connecting"`, `resetMedia()`.
3. Bind the listener set (§5.3) plus `DataReceived`.
4. `room.connect(url, token)`; rejection → `disconnectRoom("error")`.
5. Cleanup (unmount, token change, meeting leaves `STARTED`) → `alive = false` and `disconnectRoom("idle")`.

`disconnectRoom(next)` clears `roomRef` first, then `removeAllListeners()` + `disconnect()`, then sets the status and resets media state — so a late event from a dying room cannot repopulate peers. An effect-scoped `alive` flag guards the listeners and the connect rejection; the imperative paths (publish toggles, sound, mute-all) are guarded instead by `connectedRoom(roomRef)`, which requires a live room in `ConnectionState.Connected`.

## 6) Stage component — `MeetingLiveBroadcast`

### 6.1 Composition

| Piece | Role |
|---|---|
| `BroadcastVideoAttach` | Attaches one video `Track` to a `<video autoPlay playsInline muted>` filling its tile (`objectFit: cover`) |
| `BroadcastMicAttach` | Attaches one remote audio `Track` to a bare `<audio>`; applies local mute imperatively |
| `BroadcastTileAvatar` | Round avatar (org `primaryActionBackground`) with image or `FiUser` on `actionIconOnFill`; `aria-hidden` |
| `BroadcastVideoTile` | `16 / 9` tile (`asp`), org `inputBackground` + `inputBorder` + card radius, video **or** avatar placeholder, name chip pinned bottom-start |
| `BroadcastStatusSwitch` | Camera / mic / sound button: state-derived label + icon, `aria-pressed`, fixed accessible name |
| `BroadcastChairButton` | Mute-all button (no pressed state) |

Constants: `TILE_ASP = 16 / 9`, `FEATURED_AVATAR = 3.6`, `GRID_AVATAR = 2.6`.

### 6.2 Media element ownership (non-negotiable)

`BroadcastMicAttach` splits its effects on purpose:

| Effect | Dependencies | Why |
|---|---|---|
| `track.attach(el)` / `track.detach(el)` | `track` only | `attach` owns `srcObject` **and** starts playback. Re-running it when mute flips would `detach` first — which pauses the element and clears `srcObject` — cancelling the playback `startAudio()` just authorized |
| `el.muted = soundMuted` | `track`, `soundMuted` | Imperative on purpose: a React-controlled `muted` attribute fights `attach()` for the same element |

The `<audio>` element carries **no** `autoPlay` / `playsInline`: playback is owned by `attach()` plus the explicit `room.startAudio()` gesture. Video is the mirror-image case — `<video>` stays `muted` so it can autoplay frames without touching audio, and audio is never sourced from a video element.

Both attach components render `null` when `track` is `null`, so no orphan element lingers.

### 6.3 Stage layout and exclusive states

The stage is one `relative` column carrying `aria-busy` (and a `connecting` accessible label) while loading, with exactly one visual state mounted at a time:

| Condition | Rendered |
|---|---|
| `status === "connected"` | Media stack: featured chair tile + remote grid |
| `connecting` or `idle` | Centered project `Loadable` (`3rem`) — `idle` covers the token round-trip |
| `status === "error"` | `LaneFailed` (`connectionError` + `connectionErrorHint`) filling the stage |

`LaneFailed` and `Loadable` replace the media stack; they are not overlays on live tiles.

Media stack geometry: a `Flex` that is a column by default and a row from `lg`, with `direction: "ltr"` on the default breakpoint so tile order stays visual, not logical. The featured column is full width under `lg` and `50%` (`flex: 0 0 50%`) from `lg`. The remote grid is `Grid cols { default: 3, max_md: 2, max_sm: 1 }`.

Featured tile = the chairperson: `chairId` comes from the session roster (`participants` where `type === "CHAIRPERSON"`), then the peer is looked up in `local` (chair viewing their own stage) or `remotes[chairId]`. Name and avatar prefer roster values (`chair?.name`, `chair?.avatarUrl`) and fall back to the LiveKit peer name — LiveKit is a media plane, not an identity source. Grid peers = every remote except the chair, plus the local peer when the viewer is not the chair.

One `BroadcastMicAttach` is rendered per remote peer. Local audio is never played back (no self-echo).

### 6.4 Controls

A wrapping, centered row of three `BroadcastStatusSwitch` buttons — camera, mic, sound — each with:

- label from state (`videoOn` / `videoOff`, `micOn` / `micOff`, `soundOn` / `soundOff`) so an active control never reads as its own inverse action,
- a fixed accessible name naming the control (`videoAria` / `micAria` / `soundAria`) because `aria-pressed` already carries the state,
- label hidden at `SW.max_sm` (icon-only on phones; the accessible name still applies),
- enabled chrome = org `primaryActionBackground` + `primaryActionText`; otherwise `inputBackground` + `textPrimary`.

The chair mute-all pair sits in its own grouped well (`sectionAccentBackground` + `subtleDivider`) so it reads as moderation, not as a personal control, and renders only when `can.muteAllMedia` (§7).

## 7) Session capability

`MeetingLiveCapability` gains `muteAllMedia`, resolved in `resolveCan` as `isChair && status === "STARTED"` (and therefore `false` whenever linking is not READY, via `canNone()`).

It is a **UI gate only**, in the same family as `can.enterLive`: there is no `actions.muteAllMedia`, because the command travels on the LiveKit data channel rather than the Yjs session document. Reading it from the component keeps role logic in the session contract instead of a local `me?.type` check, and switches the button off the moment the meeting ends. Governance: `.cursor/rules/website-meeting-live-session.mdc`.

## 8) Shared failure chrome

`LaneFailed` (`website/src/app/ui/components/Wrong.tsx`) now accepts optional `title` / `description`, each falling back to `ui.components.wrong.laneFailed` when blank — the same override shape `Empty` already had. Consumers in this change set:

| Consumer | Copy |
|---|---|
| `MeetingLiveBroadcast` | `broadcast.connectionError` + `broadcast.connectionErrorHint` |
| `MeetingLinkingScreen` (FAILED) | `linking.failedMessage` + `linking.failedHint` |

`MeetingLinkingScreen` returns early on `linking === "FAILED"` and renders `LaneFailed` inside a `relative` full-height wrapper on the org `pageBackground` (the component is an absolute fill, so it needs a positioned parent). Consequence: the FAILED screen shows the shared error card, not the org logo block — the PENDING path (logo / name / `Loadable` / `pendingStatus`) is unchanged. No retry button is passed anywhere: neither surface has a retry entry point today.

## 9) i18n

`ui.layouts.meetingLayout.broadcast` (identical key sets in `ar.ts` and `en.ts`):

| Key | en | ar |
|---|---|---|
| `connecting` | Connecting to broadcast… | جاري الاتصال بالبث… |
| `connectionError` | Could not connect to the broadcast | تعذر الاتصال بالبث |
| `connectionErrorHint` | Live video and sound are unavailable right now. Check your connection, then reload the page. | لا يمكن عرض الفيديو ولا سماع الصوت الآن. تحقق من اتصالك بالإنترنت ثم أعد تحميل الصفحة. |
| `videoOn` / `videoOff` | Camera is on / Camera is off | الكاميرا تعمل / الكاميرا متوقفة |
| `videoAria` | Camera | الكاميرا |
| `micOn` / `micOff` | Mic is on / Mic is muted | الميكروفون يعمل / الميكروفون مكتوم |
| `micAria` | Mic | الميكروفون |
| `soundOn` / `soundOff` | Sound is on / Sound is muted | الصوت يعمل / الصوت مكتوم |
| `soundAria` | Sound | الصوت |
| `muteAllVideos` (+ `…Aria`) | Turn off all cameras (+ everyone else's) | إيقاف كاميرات الجميع (+ ما عدا كاميرتي) |
| `muteAllMics` (+ `…Aria`) | Mute all microphones (+ everyone else's) | كتم ميكروفونات الجميع (+ ما عدا ميكروفوني) |

Copy rules observed here: a toggle label states the **current state** (never the inverse action), the accessible name is a **noun** for the control, and the mute-all accessible name says "everyone else" because LiveKit excludes the sender.

`meetingLayout.linking` adds `failedHint`; `failedMessage` lost its trailing period because it is now a card **title**.

`meetingLayout.live.broadcastPlaceholder` was **deleted** from both locales: it belonged to the probe UI this change set replaced, and an unread key is drift. `meetingLayout.live` now holds `statusWaitingToStart`, `waitingLead`, `startMeeting`, `startMeetingAria` only.

## 10) Known ceilings (intentional, shipped state)

1. **Mute-all has no authority.** The receiving hook applies any well-formed command from any peer; the only gate is that the button is hidden for non-chairs (`can.muteAllMedia`). A participant who can reach the room token can publish the same payload from a console. Product decision on record: UI gating is considered sufficient for now. Real enforcement would need `roomAdmin` + `mutePublishedTrack` on the backend, or a signed command.
2. **Mute-all is one-shot and cooperative.** A muted peer can immediately re-enable their camera/mic; there is no locked state, no "muted by chair" indicator, and no re-apply for someone who joins after the command.
3. **`error` is terminal for the mounted stage.** `Disconnected` or a failed `connect` sets `error` and the UI offers copy only ("reload the page"). The token hook retries **network** failures while `STARTED`, but a rejected token (`error`) does not retry, and a disconnected room is not re-created without a new token identity.
4. **`unauthorized` is sticky until the next attempt.** A denied camera/mic permission shows the off state; there is no browser-permission recovery guidance, and the status only changes when the user presses the control again.
5. **Sound is off by default and needs a gesture** (browser autoplay policy). If the browser refuses `startAudio()`, the switch falls back to muted with no explanatory copy.
6. **A source-muted microphone is indistinguishable in the UI.** If the OS or headset mutes the device, the publication still exists and stays attached, so remote peers see "mic on" and hear silence. Nothing in the app can detect or fix that — it is an operator-side check during incident triage.
7. **The grid is unbounded.** Every remote peer gets a tile (3 / 2 / 1 columns); there is no pagination, virtualization, or subscription throttling, so a large meeting scales linearly in tiles and subscribed tracks.
8. **Every roster member gets LiveKit's default grant** (publish + subscribe). `MeetingParticipant.type` is not mapped to publish permissions, so a `VIEWER` can publish media as far as LiveKit is concerned.
9. **Media join is not attendance.** Connecting to the room does not write `attended_at` / `left_at`; attendance stays on the session/socket path.
10. **In-process `STARTED` peek** ceiling is inherited from the backend controller: a Node process without the live registry entry answers `MEETING_NOT_LIVE`, which the hook surfaces as broadcast `error` (`../backend/contracts/livekit-media-plane.md` §6.4).

## 11) Failure modes

| Scenario | Behavior |
|---|---|
| Session not `STARTED` / missing route ids | Token `idle` → broadcast `idle` → `Loadable` (component is only mounted while `STARTED`, so this is a transient frame) |
| Token network failure | Token stays `pending`, quiet 3s retry (no toast) → stage keeps loading |
| Token rejected (`NOT_VALID_CREDENTIAL`, `MEETING_NOT_LIVE`) | Token `error` → broadcast `error` → `LaneFailed` |
| Response missing `url` | Union keeps `pending`; `Room.connect` is never called with a partial payload |
| `Room.connect` rejects (bad/expired JWT, unreachable host) | `disconnectRoom("error")` → `LaneFailed` |
| Server or network drops the room | `ConnectionState.Disconnected` → `error` (LiveKit's own reconnect attempts surface as `connecting` first) |
| Camera / mic permission denied | That control's status → `unauthorized`; the other controls keep working |
| Browser blocks audio playback | `soundStatus` reverts to `muted` |
| Peer mutes camera | Publication `isMuted` → `videoTrack: null` → avatar tile (no black or white frame) |
| Peer mutes mic | Track stays attached; audio goes silent without tearing down playback |
| Chair not connected to the room | Featured tile renders the roster avatar and name |
| Meeting leaves `STARTED` / component unmounts | Effect cleanup disconnects the room and resets media state to `muted` |

## 12) Traceability map (this change set)

### `backend/`

| Path | State | Where described |
|---|---|---|
| `src/app/http/controllers/website/custom/MeetingLiveKitTokenController.ts` | modified — `Result` adds `url`; step 7 returns `LiveKitHelper.clientUrl()` | §4; `../backend/contracts/livekit-media-plane.md` §6.2–§6.3 |
| `src/app/helpers/LiveKitHelper.ts` | unchanged — `clientUrl()` already existed and is now consumed | `../backend/contracts/livekit-media-plane.md` §5.3 |
| `lib/**`, `.types/**`, `.exporters/**`, `.webpack_root.ts` | generated build / autoload output present in the working tree; not behavioral, not narrated | — |

### `website/`

| Path | State | Where described |
|---|---|---|
| `package.json` | modified — `livekit-client@2.21.0` exact pin | §3 |
| `yarn.lock` | modified — client + transitive entries | §3 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitRoom.ts` | **added** — central room hook (status, peers, publish, sound, mute-all, lifecycle) | §5 |
| `src/app/ui/components/meeting/MeetingLiveBroadcast.tsx` | modified — probe UI replaced by the real stage, attach components, controls, chair group | §6 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveKitToken.ts` | join fetch — idle unless `can.enterLive && STARTED` + route ids; union `{ status, token, url }` | §4 |
| `src/app/ui/components/meeting/hooks/useMeetingLiveSession.tsx` | modified — `muteAllMedia` capability (type, `canNone`, `resolveCan`) | §7 |
| `src/app/ui/components/meeting/MeetingLinkingScreen.tsx` | modified — FAILED early return renders `LaneFailed`; PENDING path unchanged | §8 |
| `src/app/ui/components/Wrong.tsx` | modified — `LaneFailed` optional `title` / `description` | §8 |
| `src/resources/translations/ar.ts` | modified — `broadcast.*` added, `linking.failedHint` added, `failedMessage` punctuation, `live.broadcastPlaceholder` removed | §9 |
| `src/resources/translations/en.ts` | modified — same key set | §9 |
| `src/app/ui/pages/Meeting.tsx` | unchanged — already the `STARTED` mount owner | §2 |
| `lib/tsconfig.tsbuildinfo` | generated by `yarn type-check`; not narrated | — |

### Workspace root (`docs/` / `.cursor/`)

| Path | State | Where described |
|---|---|---|
| `docs/platforms/website/flow-meeting-broadcast.md` | **added** — this page | — |
| `docs/platforms/backend/contracts/livekit-media-plane.md` | modified — `{ token, url }`, hook union, probe removal, client consumer | backend contract |
| `docs/platforms/backend/contracts/client-portal-http-website.md` | modified — response shape + hook line | HTTP index |
| `docs/platforms/backend/modules/runtime-integrations.md` | modified — §7b website client row | integration index |
| `docs/platforms/backend/README.md` | modified — LiveKit contract row mentions the client | backend index |
| `docs/platforms/website/organization-host-routing.md` | modified — §5.3 FAILED chrome + capability row, §5.5 broadcast body, §8 limits, §10 / §10m, §11 | shell contract |
| `docs/platforms/website/README.md` | modified — flow index + change-set pointer | website index |
| `docs/platforms/website/component-structure.md` | modified — meeting shell component list | component index |
| `docs/platforms/website/shared-ui-and-shell.md` | modified — `LaneFailed` override props | shared UI |
| `docs/invariants/website.md` | modified — W60 media element ownership | invariants |
| `docs/README.md` | modified — website doc index row | root index |
| `.cursor/rules/website-meeting-livekit-broadcast.mdc` | **added** — durable broadcast invariants | governance |
| `.cursor/rules/livekit-media-plane.mdc` | modified — response + hook API, probe removed | governance |
| `.cursor/rules/website-meeting-live-session.mdc` | modified — FAILED chrome + `muteAllMedia` gate-only | governance |
| `.cursor/rules/website-meeting-shell.mdc` | modified — pointer: broadcast internals live in the new rule | governance |
| `.cursor/skills/website-meeting-broadcast/SKILL.md` | **added** — broadcast extension workflow | governance |
| `.cursor/skills/meeting-livekit-token/SKILL.md` | modified — `{ token, url }`, probe removed, hand-off to broadcast skill | governance |
| `backend` / `website` gitlinks | modified pointers only — no root behavior | — |

## 13) Verification

- `yarn type-check` in `website/` and in `backend/` (the only checks that exist for these paths).
- Two browsers on one `STARTED` meeting: each side sees the other's tile; camera off shows the avatar tile, not a blank frame.
- Sound button is off on load; pressing it starts audio and the label reads the on state. A refused autoplay leaves it off.
- Mic on with an OS-muted device: remote shows mic on and hears silence (§10.6) — check the device before the code.
- Chair mute-all cameras / mics: other peers' publications stop; the chair's own media is untouched; the buttons are absent for non-chairs and disappear when the meeting ends.
- Kill the LiveKit connection: stage shows the failure card, not a spinner. Reject the token (leave `STARTED`): same card.
- Break the socket handshake instead: `MeetingLinkingScreen` FAILED shows the shared error card with the linking copy.

## 14) Related

- `organization-host-routing.md` §5.3–§5.5, §8, §10m — session, shell, limits, change-set inventory
- `../backend/contracts/livekit-media-plane.md` — helper, join-token HTTP, participant JWT cache
- `../backend/contracts/meeting-participant-domain.md` §3.7 — `livekit_*` columns (never on GQL)
- `../backend/contracts/meeting-realtime-socket.md` — the socket plane the session rides on
- `shared-ui-and-shell.md` — `Empty` / `LaneFailed` chrome
- `docs/invariants/website.md` W59, W60
- `.cursor/rules/website-meeting-livekit-broadcast.mdc`, `.cursor/rules/livekit-media-plane.mdc`, `.cursor/rules/website-meeting-live-session.mdc`
- `.cursor/skills/website-meeting-broadcast/SKILL.md`, `.cursor/skills/meeting-livekit-token/SKILL.md`
