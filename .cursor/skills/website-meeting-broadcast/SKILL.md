---
name: website-meeting-broadcast
description: >-
  Ships or extends the organization-host Meeting broadcast client:
  useMeetingLiveKitRoom (Room.connect, peer projection, camera/mic publish,
  local sound autoplay gate, cooperative mute-all data channel) and
  MeetingLiveBroadcast (featured chair tile + remote grid, track attach
  components, control switches, chair mute-all on can.muteAllMedia, non-chair
  raise-hand control, tile queue/floor badges and ranked peer order). Use when
  changing LiveKit client behavior on website/, adding a media control or tile
  state, debugging silent audio or a black tile, or wiring a new data-channel
  command. For the join-token HTTP path use meeting-livekit-token; for shell
  drawer/header IA use website-meeting-shell; for linking/can/actions use
  website-meeting-live-session; for talk-queue ordering rules use
  website-meeting-talk-queue.
---

# Website Meeting broadcast

## When to Use

- Editing `MeetingLiveBroadcast.tsx` or `hooks/useMeetingLiveKitRoom.ts`.
- Adding a media control, tile state, or placeholder to the broadcast stage (including live-map state chrome such as the talk-queue badges and the raise-hand control).
- Adding a data-channel command, or changing mute-all behavior / its gate.
- Debugging "video works, no audio", a black/blank tile, an eternal spinner, or a control whose label contradicts its state.
- Changing the join payload consumed by the room (`{ token, url }`).

## Read first

- `docs/platforms/website/flow-meeting-broadcast.md` (whole page — §5 hook, §6 stage, §10 ceilings, §11 failure modes)
- `.cursor/rules/website-meeting-livekit-broadcast.mdc`
- `docs/invariants/website.md` W60 (media element ownership)
- `docs/platforms/backend/contracts/livekit-media-plane.md` §6 (token contract)
- `docs/platforms/website/organization-host-routing.md` §5.3, §5.5 (session capability, mount ownership)

## Instructions

1. **Locate the state, don't duplicate it.** All room/media state lives in `useMeetingLiveKitRoom`. Add a field to that hook's return type; never hold a `Room`, `Track`, or connection flag inside the component or a page.
2. **Keep the token contract intact.** `Room.connect(url, token)` runs only for `status === "ready"`. If you extend the join payload, extend the union in `useMeetingLiveKitToken` (ready ⇒ all values present) and the backend `Result` together, and keep `error` propagating to broadcast `error`.
3. **Touching an attach component?** The attach effect depends on the track only; anything else (mute, volume, sink id) is applied imperatively in a separate effect. Do not add `autoPlay` to `<audio>`, and do not convert `el.muted` into a JSX attribute (W60).
4. **New publish control:** extend `MeetingLiveKitPublishKind` + `setPublishEnabled` so the success/reject → `enabled` / `muted` / `unauthorized` mapping and the `syncPeers` call stay in one place.
5. **New playback control:** model it like `soundStatus` — default off, SDK start call inside the click stack, revert on rejection. Never merge a playback control with a publish control.
6. **New data-channel command:** add a `CMD_*` constant, publish with `{ reliable: true }`, decode in `onDataReceived`, ignore unknown payloads. Gate the sender UI on a session `can.*` capability and state plainly in docs that the receiver applies it without authority (or implement server enforcement instead).
7. **New capability gate:** add it in `useMeetingLiveSession` (type + `canNone()` + `resolveCan`) and read `can.*` in the component. Gate-only capabilities have no `actions` entry — say so in the docs row.
8. **Stage states stay exclusive:** media stack / `Loadable` / `LaneFailed`. Reuse shared chrome with copy overrides instead of hand-rolling an icon + message, and remember `LaneFailed` is an absolute fill (parent must be `relative`).
9. **Copy:** add both `ar` and `en` under `ui.layouts.meetingLayout.broadcast` with identical key sets. A toggle label states the current state; the accessible name is the control noun; a mute-all name says "everyone else" (LiveKit excludes the sender).
10. **Identity comes from the session roster** (`meeting.participants`), with the LiveKit peer name as fallback only. LiveKit is a media plane, not an identity or attendance source.
11. **Live-map state on the stage** (talk queue today): signal it with tile chrome and an action-row control — never by swapping the featured chair tile, and never by changing publish/mute state. Tile state badges are `role="img"` with a label; decorative rails are `aria-hidden`. If you order peers by that state, use the same total comparator (ordering key + stable id tie-break) as the page that owns the state. Contract: `.cursor/rules/meeting-talk-queue.mdc`, skill `website-meeting-talk-queue`.
12. **Document ceilings honestly.** If the change leaves a gap (no enforcement, no retry, no recovery), add it to `flow-meeting-broadcast.md` §10 and the `organization-host-routing.md` §8 limit list rather than implying it works.
13. **Verify with existing scripts only:** `yarn type-check` in `website/` (and `backend/` when the controller changed). Then the two-browser pass in `flow-meeting-broadcast.md` §13 — including the OS-level mic-mute case before suspecting the code.
