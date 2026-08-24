# CPanel Meeting Management

Read-only supervisor **meetings** directory (platform-wide, not `me: true`). No `live_state` in the client selection.

Backend: `docs/platforms/backend/contracts/supervisor-catalog-and-home.md` §5.

## 1) Routes

| Identify | Path |
|---|---|
| `SupervisorMeetings` | `/supervisor/meetings` |
| `SupervisorMeeting` | `/supervisor/meetings/:id` |

Register **meetings** before `:id`. Drawer: `FiCalendar` immediately after Customers. Nested params.

## 2) Adapters

List: `"supervisor-meetings"`, `listable: "meetings"`, history search key `"meetings"` (subject iLike). No status chip filter on the directory. `meetingStats` (`total_count`, `started_count`) on `Stats`.

Detail: `section.meeting`. `PageStateLane` overlay flags on `useSupervisorMeeting`. Contract: `docs/platforms/cpanel/supervisor-state-lanes.md`.

## 3) UI

`SupervisorMeetingCard` + `StatusChip` for meeting status (and org status where shown). Same chip language as customer detail org status.

Chairperson from slim `_Member`. Organization + customer nested as in the hook selection sets.

Home started/waiting cards reuse this card; they are **not** this directory’s adapter.

## 4) Traceability

| Path | Role |
|---|---|
| `cpanel/.../meetings/*` | List/detail/card |
| `cpanel/.../StatusChip.tsx` | Shared status pills |
| `cpanel/.../pages/supervisor/SupervisorMeetings.tsx` | Thin list |
| `cpanel/.../pages/supervisor/SupervisorMeeting.tsx` | Thin detail |
| `backend/.../MeetingBridge.ts` | GQL; `expect: live_state` |
| `backend/.../MeetingStatsBridge.ts` | Stats |
| `backend/.../MemberBridge.ts` | Chairperson nest |

## Related

- `docs/platforms/cpanel/supervisor-state-lanes.md`
- `.cursor/skills/cpanel-supervisor-read-directory/SKILL.md`
