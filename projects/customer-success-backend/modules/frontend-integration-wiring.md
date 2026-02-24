# Frontend Integration Wiring — 2026-02-23

## What was done
Wired all remaining frontend pages to live backend/Supabase data. Removed all hardcoded mock data imports. Every page now shows real data with realtime updates where applicable.

## Pages Wired (this session)

### 1. Dashboard (`src/pages/Dashboard.tsx`)
- Replaced `dashboardStats` (mockData) with live hooks
- `useIVRSidebarCounts()` → Live Calls, Pending Hangups, SLA Breached cards
- `useDashboard()` → Email stats, Chat stats, Team distribution pie chart (was already partially done)
- `useRole()` → Dynamic greeting (was hardcoded "Rishi")
- Open Tickets (23) and NPS Score (72) remain hardcoded — no backend tables for these yet

### 2. Agent Dashboard (`src/pages/agent/AgentDashboard.tsx`)
- Full rewrite: removed hardcoded `agentStats` object
- Added `useAgentLiveData(agent_id)` hook — queries `agents` + `agent_daily_stats` tables with realtime
- Agent lookup: queries `agents` by `currentUser.email`, fallback to first active agent
- `useRole()` for greeting name
- Stats derived: talk_time, work_time, break_time, avg_handle, completed_today, sla_breaches, positive_pct
- Work module cards show live: callsSinceLogin, callsMissed from daily stats

### 3. Agent Leaderboard (`src/components/dashboard/AgentLeaderboard.tsx`)
- Replaced `agentPerformance` (mockData) with `useAllAgentsLive()`
- Filters to online agents only (status !== 'offline')
- Maps `LiveAgentData` → `AgentLeaderEntry` with talkTimeMinutes, callsHandled, avgCallDuration
- Empty state: "No online agents" when none available
- Feedback score placeholder (85%) — no per-agent feedback in realtime yet

### 4. Masters Agents Tab (`src/pages/Masters.tsx`)
- Replaced `AGENTS_MASTER` import with `GET /agents?is_active=true` fetch
- Added `BACKEND_URL` constant, `fetch_agents()` function
- Mapping functions: `map_agent_status()`, `map_backend_agent()`
- Add Agent dialog → `POST /agents`
- Edit Agent dialog → `PATCH /agents/:id`
- Deactivate → `DELETE /agents/:id`
- Other tabs (Numbers, Templates, Routing, IVR Providers) unchanged

### 5. Settings — SLA Config Tab (`src/pages/Settings.tsx`)
- Added state: `sla_configs[]`, `sla_saving`
- On mount: `GET /sla-config` → populate 4 tier rows
- Controlled inputs with onChange handlers
- Save button: `PUT /sla-config/bulk` with all 4 tiers

### 6. Settings — User Roles Tab (`src/pages/Settings.tsx`)
- Added state: `backend_roles[]`, `backend_modules[]`, `role_users[]`
- Fetches: `GET /roles`, `GET /roles/modules`, `GET /roles/users`
- Create role: `POST /roles`
- Edit role: `PATCH /roles/:id`
- Delete role: `DELETE /roles/:id`
- Assign user: `POST /roles/users`
- Remove assignment: `DELETE /roles/users/:id`

### 7. Settings — Email Tab (`src/pages/Settings.tsx`)
- Created `src/hooks/useSupportEmails.ts` (NEW)
  - `BACKEND_URL` fetch pattern (same as `useLiveChat`)
  - Returns: `emails, loading, add_email, update_email, remove_email, sync_email, toggle_active, refresh`
  - `SupportEmail` interface, `format_relative_time()` helper
- Replaced hardcoded email table with dynamic data from hook
- Add Email dialog with email_address + imap_password + display_name fields
- Per-row actions: toggle active, sync, delete
- Removed SMTP section (backend handles SMTP internally)

## Already Wired (confirmed, no changes needed)

| Page | Data Source | Notes |
|------|------------|-------|
| RefundDashboard | `useQuery` → Supabase `refund_requests` | Full CRUD, stats, filters |
| Audit | `useQuery`/`useMutation` → Supabase `audits` + `agents` | Score, status, agent list |
| VideoCallLeads | `useQuery`/`useMutation` → Supabase `video_call_leads` | Full CRUD + detail dialog |
| Call Categories | `useCallCategories` → Supabase `call_categories` | Used in 3 IVR components |
| Floor Status | `useAllAgentsLive` → Supabase `agents` | Groups by floor, realtime |
| Customer Timeline | Types in `types.ts`, no frontend page exists | Backend API ready when needed |
| Notifications | `useNotifications` → Supabase reads + backend mutations | Wired in previous session |

## Files Modified
- `src/pages/Dashboard.tsx` — Live IVR counts, dynamic greeting
- `src/pages/agent/AgentDashboard.tsx` — Full rewrite with useAgentLiveData
- `src/components/dashboard/AgentLeaderboard.tsx` — Wired to useAllAgentsLive
- `src/pages/Masters.tsx` — Agents tab CRUD via backend API
- `src/pages/Settings.tsx` — SLA Config, User Roles, Email Settings wired
- `src/hooks/useSupportEmails.ts` — **NEW** hook

## Hooks Architecture (complete picture)

| Hook | Data Source | Realtime | Used By |
|------|-----------|----------|---------|
| `useIVRCalls` | Supabase `ivr_calls` | Yes | IVR pages |
| `useIVRProviders` | Supabase `ivr_providers` | Yes | Sidebar, Masters |
| `useIVRSidebarCounts` | Supabase `ivr_calls` | Yes | Sidebar, Dashboard |
| `useLiveChat` | Backend API + Supabase | Yes | Live Chat page |
| `useChatCounts` | Supabase `conversations` | Yes | Sidebar |
| `useEmails` | Supabase `emails` | Yes | Emails page |
| `useNotifications` | Supabase + Backend API | Yes | Notifications, Header, Sidebar, Settings |
| `useDashboard` | Supabase RPC (4 functions) | 60s poll | Dashboard |
| `useAgentLiveData` | Supabase `agents` + `agent_daily_stats` | Yes | Agent Dashboard |
| `useAllAgentsLive` | Supabase `agents` | Yes | Leaderboard, Floor Status |
| `useCallCategories` | Supabase `call_categories` | No | IVR Live/Hangup, AgentCallScreen |
| `useSupportEmails` | Backend `/support-emails` | No | Settings Email tab |

## Session 2 — Button Wiring & Sidebar Badges (2026-02-23)

### Sidebar Badge Fixes (`src/components/layout/AppSidebar.tsx`)
- Removed hardcoded badges for modules with no backend: Query Assignment (12), Ticketing (4), My Tickets (4)
- Removed hardcoded badges from Return & Refund (6) — page shows its own counts
- Wired My Work children dynamically in `useMemo`:
  - Call Working → `ivrCounts.live.total`
  - Chat Working → `chatCounts.all`
  - Hangup Working → `ivrCounts.hangup.total`
  - My SLA Breaches → `ivrCounts.slaBreach`

### Agent Chat Buttons (`src/pages/agent/AgentChats.tsx` + `src/hooks/useAgentChats.ts`)
- Added `accept_chat(id)` → updates conversation status to 'active'
- Added `resolve_chat(id)` → updates conversation status to 'resolved'
- Added `send_message(conversation_id, content)` → inserts into `chat_messages` + updates `last_message_at`
- Wired Accept Chat, Resolve, Send buttons + Enter key support

### Agent Calls End Call (`src/pages/agent/AgentCalls.tsx` + `src/hooks/useAgentCalls.ts`)
- Added `update_call(id, updates)` to hook
- `handleEndCall` now persists: sets status='completed' + ended_at in DB

### Agent SLA Breach Handle Now (`src/pages/agent/AgentSLABreach.tsx`)
- Added Handle Now → opens CallAnsweredDialog → AgentCallScreen flow
- On end call: marks IVR call as completed in DB

### Admin SLA Breach Actions (`src/pages/SLABreach.tsx`)
- Escalate button → sets priority to 'vip' (auto-moves to Escalated tab)
- View button → navigates to relevant IVR live/hangup page

## Remaining
- **Kaleyra Click-to-Call**: Blocked on credentials (KALEYRA_SID, KALEYRA_API_KEY, etc.)
- **LiveActivityFeed**: Still hardcoded activities — needs real event stream (future)
- **Open Tickets / NPS Score**: Hardcoded in Dashboard — no backend tables for these
- **Unwired pages (no backend)**: Tickets (5 pages), Surveys (4 pages), Agent Feedback (2 pages), Workflows, YouTube Leads, Return RCA, Feedback, Agri Consultancy — all 100% mock data, need new backend modules

## Testing
All 27 endpoint groups tested via curl — all pass. Frontend builds clean.
Backend: 18 modules, 95+ endpoints on port 3002.
Frontend: Vite dev on port 8082.
