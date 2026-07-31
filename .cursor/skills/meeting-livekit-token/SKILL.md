---
name: meeting-livekit-token
description: >-
  Ships or extends the organization-host LiveKit join-token path:
  MeetingLiveKitTokenController on POST /website/custom/org/livekit_token
  (org_host + peek STARTED + MeetingParticipant reuse-or-mint) and website
  useMeetingLiveKitToken (union { status, token, url }, quiet network retry).
  Use when changing join-token HTTP, participant livekit_* columns, the fetch
  hook, or skipNetworkToast. For the browser room/stage that consumes the
  token, use website-meeting-broadcast.
---

# Meeting LiveKit join token

## When to Use

- Changing `MeetingLiveKitTokenController`, `POST /custom/org/livekit_token`, or `API.CUSTOM.ORG_LIVEKIT_TOKEN`.
- Changing `MeetingParticipant.livekit_token` / `livekit_token_expires_at` reuse-or-mint.
- Changing or mounting `useMeetingLiveKitToken`.
- The room/stage that consumes the token is a separate surface: `.cursor/skills/website-meeting-broadcast/SKILL.md`.

## Read first

- `docs/platforms/backend/contracts/livekit-media-plane.md` §6–§9
- `.cursor/rules/livekit-media-plane.mdc`
- `.cursor/rules/meeting-participant-roster.mdc` (token columns never on GQL)
- `docs/platforms/backend/contracts/client-portal-http-website.md` (org LiveKit token surface)
- For session `STARTED` / linking: `.cursor/skills/website-meeting-live-session/SKILL.md`
- For LiveKit helper mint/room APIs only: `livekit-media-plane.md` §5

## Instructions

1. **Route** stays on `OrgCustomRouter` with **per-route** `middleware("org_host")`. Do not put `org_host` on `/custom/org/start`. Do not mint tokens from GraphQL.
2. **Controller name** is `MeetingLiveKitTokenController` (meeting domain). Do not rename to `OrgLiveKitTokenController` just because the path is under `/org`.
3. **Authz order** matches the controller steps: body credentials → member `id`+`access_token` → meeting under `currentOrganization` + member org match → roster row → `peekMeetingLiveDoc` + `readLiveFields` `STARTED` (never `getOrCreate` / never create for this gate).
4. **Reuse-or-mint:** reuse when `livekit_token_expires_at` ≥ now+6h; else `LiveKitHelper.createAccessToken` with `identity` = member id string; persist token + expiry `now+12h` aligned with `LiveKitHelper.DEFAULT_ACCESS_TOKEN_TTL`. One JWT per roster member — never share across members.
5. **Response** is `{ token, url }` — `url` from `LiveKitHelper.clientUrl()`, recomputed per response and never persisted next to the cached JWT. The client has no LiveKit env key, so any new client-needed value ships on this response.
6. **Website hook** public API is a union on `status` (`idle|pending|ready|error`): `ready` carries `token` **and** `url`, every other status carries neither — a partial payload must degrade to `pending`, never reach `Room.connect`. Gate fetch on `useMeetingLiveSession().meeting.status === "STARTED"` plus route ids. Quiet network retry uses `skipNetworkToast` (declared in `website/src/types/extends/global.ts`, honored in `axios.ts`) — do not cast Axios config. Sole consumer is `useMeetingLiveKitRoom`.
7. **Do not** expose `livekit_*` on GQL SDL/bridges. **Do not** invent checked-in SQL migrations; use the repo’s ORM alter/sync.
8. After controller rename/add, let backend boot refresh gitignored `backend/.types/controllers.ts` (or clear incremental `tsbuildinfo` if type-check lags).
9. Verify with existing scripts only: `yarn type-check` in `backend/` and `website/`.
