# IVR Pages — Supabase Wiring & Sidebar Counts

## Date: 2026-02-23

## What was done
Wired all 7 IVR pages from hardcoded mock data (`src/data/ivrData.ts`) to live Supabase `ivr_calls` table. Created unified hook, real-time subscriptions, sidebar badge counts.

### 1. Unified Hook — `useIVRCalls.ts`
- Created `src/hooks/useIVRCalls.ts` — single hook powering all 7 IVR pages
- Filter params: `{ statuses?, department?, slaOnly?, agentId?, limit?, orderBy? }`
- Returns: `calls`, `loading`, `stats`, `updateCall`, `assignCall`, `bulkAssign`, `logAttempt`, `endCall`, `refresh`
- `mapRow()` converts DB snake_case → camelCase frontend (`IVRCallRow`)
- `toSnake()` for write operations (camelCase → snake_case)
- `computeStats()` — client-side stats: total, waiting, active, completed, missed, hangup, slaBreached, avgDuration
- Real-time subscription via `postgres_changes` on `ivr_calls`
- `IVRCallRow` extends `IVRCall` with extra DB fields: `isSlaBreached`, `action`, `remarkByReceiver`, `entryDate`

### 2. Sidebar Counts — `useIVRSidebarCounts.ts`
- Created `src/hooks/useIVRSidebarCounts.ts` — lightweight hook for sidebar badges
- Single query fetches `department, status, is_sla_breached` from all `ivr_calls`
- Groups by:
  - **Live**: statuses `['waiting', 'ringing', 'missed']` — calls needing attention (NOT `active` — already handled)
  - **Hangup**: status `'hangup'` per department
  - **SLA Breach**: `is_sla_breached = true`
  - **allTotal**: total count of all calls (for admin view)
- Real-time subscription on `ivr-sidebar-counts` channel
- `AppSidebar.tsx` wired via `useMemo` — same pattern as `useChatCounts`

### 3. Pages Wired

| Page | Route | Hook Params | Mutations |
|------|-------|-------------|-----------|
| CallHistory | `/call-history` | `{ statuses: ['completed','missed','hangup'], limit: 500 }` | None |
| IVRConsolidated | `/ivr-live/admin` | `{ limit: 500 }` (all statuses) | None |
| SLABreach | `/sla-breach` | `{ slaOnly: true, limit: 500 }` | None |
| IVRLiveDepartment | `/ivr-live/:department` | `{ statuses: ['waiting','ringing','active','missed'], department }` | `updateCall`, `endCall` |
| IVRLive | `/ivr-live` (active call screen) | `{ statuses: ['waiting','ringing','active'], limit: 50 }` | `endCall` |
| IVRHangup | `/ivr-hangup` | `{ statuses: ['hangup'] }` | `assignCall`, `bulkAssign`, `logAttempt`, `endCall` |
| IVRHangupDepartment | `/ivr-hangup/:department` | `{ statuses: ['hangup'], department }` | `assignCall`, `bulkAssign`, `logAttempt`, `endCall` |

### 4. Adapter Functions
- `toHangupCall()` in IVRHangup.tsx — converts `IVRCallRow` → `HangupCall` type:
  - Derives status from DB fields (action, attempts, isSlaBreached, agentId)
  - Maps priority (vip/high → 'high', repeat → 'repeat_caller')
  - Builds synthetic attempts array from count
- `callToBreach()` in SLABreach.tsx — converts to breach display format with `deriveSeverity()` and `deriveBreachStatus()`

### 5. Key Gotcha — Sidebar Count Mismatch (Fixed)
- **Problem**: Sidebar initially used `['waiting', 'ringing', 'active']` for live counts. But department pages show queue (waiting+ringing) + missed — NOT active calls.
- **Result**: General showed 4 in sidebar but 3 on page (1 active counted by sidebar but not in queue). Retailer showed 1 but page had 2 (missed call not counted).
- **Fix**: Changed `LIVE_STATUSES` to `['waiting', 'ringing', 'missed']` — counts calls needing attention, not those already being handled.

### 6. IVRConsolidated Stats Fix
- "Live Calls" was counting everything non-hangup (including completed/missed) → fixed to `waiting + active` only
- "Waiting" was `waiting + hangup` → fixed to just `waiting`

## Files Created
- `src/hooks/useIVRCalls.ts`
- `src/hooks/useIVRSidebarCounts.ts`

## Files Modified
- `src/pages/CallHistory.tsx`
- `src/pages/IVRConsolidated.tsx`
- `src/pages/SLABreach.tsx`
- `src/pages/IVRLiveDepartment.tsx`
- `src/pages/IVRLive.tsx`
- `src/pages/IVRHangup.tsx`
- `src/pages/IVRHangupDepartment.tsx`
- `src/components/layout/AppSidebar.tsx`

## What Stays Mock
- `mockCustomer`, `mockOrders`, `mockInteractions`, `solutionScripts` in IVRLive.tsx (no customers/orders tables yet)
- `DEPARTMENT_LABELS`, `DEPARTMENT_CATEGORIES` from ivrData.ts (static config)
- Agent hooks (`useAgentCalls`, `useAgentHangups`, etc.) — separate, serve Agent Dashboard

## DB Notes
- `ivr_calls` table has 30+ columns, `agent_id` is UUID type (not string IDs like 'ag-1')
- Test data seeded: ~21 calls across departments (general, retailer, app, kyc, international) with mixed statuses
