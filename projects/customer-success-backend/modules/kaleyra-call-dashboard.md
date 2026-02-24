# Kaleyra Call Dashboard — Sync + Display Call Data

**Date:** 2026-02-24
**Status:** Backend complete, tested, verified with live data

## What Was Built

Kaleyra Voice already had click-to-call + callback endpoints. This session added:
1. **Supabase table** `kaleyra_call_logs` — stores synced call data from Kaleyra API
2. **4 Supabase RPC functions** — dashboard aggregations (call volume, agent stats, daily trend, status breakdown)
3. **Sync service** — pulls call logs from Kaleyra `method=dial` API, paginates, upserts into Supabase
4. **5 new endpoints** — sync trigger + 4 dashboard queries

## Architecture

```
Kaleyra Voice API (api-voice.kaleyra.com)
  → POST /calls/sync (manual trigger)
    → Fetches method=dial with fromdate/todate, paginated (200/page)
    → Upserts into kaleyra_call_logs (conflict on id)

kaleyra_call_logs (Supabase)
  → 4 RPC functions (get_kaleyra_call_volume, agent_stats, daily_volume, status_breakdown)
    → Called by dashboard endpoints via Promise.all
```

## Supabase Table: `kaleyra_call_logs`

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PK | Kaleyra call ID |
| `uniqid` | TEXT | Kaleyra internal unique ID |
| `callfrom` | TEXT NOT NULL | Customer/caller number |
| `callto` | TEXT NOT NULL | DID (8068921234) or agent number |
| `start_time` | TIMESTAMPTZ NOT NULL | Call start |
| `end_time` | TIMESTAMPTZ | Call end |
| `duration` | INTEGER | Total duration (seconds) |
| `billsec` | INTEGER | Billable/talk duration (seconds) |
| `status` | TEXT NOT NULL | ANSWER, NOANSWER, BUSY, CANCEL, FAILED, etc. |
| `location` | TEXT | Caller state (BIHAR, UP EAST, etc.) |
| `provider` | TEXT | Telecom (JIO, AIRTEL, IDEA, Tata Teleservices) |
| `service` | TEXT NOT NULL | Incoming, CallForward, Click2Call |
| `caller_id` | TEXT | DID used |
| `recording` | TEXT | Recording URL (play.solutionsinfini.com) |
| `synced_at` | TIMESTAMPTZ | When this record was synced |

**Indexes:** start_time, service, callto, status
**RLS:** Allow all for service role

## RPC Functions

| Function | Params | Returns |
|----------|--------|---------|
| `get_kaleyra_call_volume(p_start, p_end)` | TIMESTAMPTZ range | `{total, incoming, forwarded, click2call}` |
| `get_kaleyra_agent_stats(p_start, p_end)` | TIMESTAMPTZ range | `[{agent_number, calls_answered, total_talk_seconds, missed_calls}]` |
| `get_kaleyra_daily_volume(p_start, p_end)` | TIMESTAMPTZ range | `[{date, incoming, forwarded}]` — IST timezone grouping |
| `get_kaleyra_status_breakdown(p_start, p_end)` | TIMESTAMPTZ range | `[{status, count}]` |

Agent stats filters to `service = 'CallForward'` (agent legs only).

## New Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/calls/sync` | Sync Kaleyra → Supabase. Body: `{from_date?, to_date?}` (YYYY/MM/DD). Default: today |
| GET | `/calls/dashboard/overview` | All metrics parallel (Promise.all) |
| GET | `/calls/dashboard/agent-stats` | Per-agent performance |
| GET | `/calls/dashboard/daily-volume` | Daily trend (IST) |
| GET | `/calls/dashboard/call-list` | Paginated call log. Filters: service, agent_number, page, limit |

All dashboard endpoints accept: `?range=today|7d|30d|custom&start_date=&end_date=`

## Files Modified

| File | Change |
|------|--------|
| `src/kaleyraVoice/dto/callDashboard.dto.ts` | **NEW** — CallDashboardQueryDto, SyncCallLogsDto, 7 response DTOs |
| `src/kaleyraVoice/kaleyraVoice.service.ts` | Added sync_call_logs(), get_dashboard_overview(), get_agent_stats(), get_daily_volume(), get_call_list() + RPC wrappers + resolve_dates() + parse_kaleyra_datetime() |
| `src/kaleyraVoice/kaleyraVoice.controller.ts` | 5 new routes |
| Supabase migration `create_kaleyra_call_logs` | Table + indexes + RLS |
| Supabase migration `create_kaleyra_dashboard_rpcs` | 4 RPC functions |

## Test Results (2026-02-24)

- **Sync today:** 576 records synced
- **Sync 7 days (Feb 18-24):** 2088 records synced
- **Call volume (7d):** 1231 incoming, 825 forwarded, 32 click2call
- **Agent stats (7d):**
  - 9201952685: 34 answered, 4532s talk, 103 missed
  - 9201972062: 22 answered, 3188s talk, 112 missed
  - 9238109518: 20 answered, 4419s talk, 47 missed
  - 6264437618: 19 answered, 1608s talk, 93 missed
  - 9201836530: 4 answered, 590s talk, 77 missed
  - 9201972072: 0 answered, 0s talk, 96 missed
- **Status breakdown (7d):** ANSWER 1233, NOANSWER 313, ANSWERED-B 190, FAILED 138, CANCEL 88
- **Recordings:** Present on answered CallForward legs (play.solutionsinfini.com URLs)
- **Location/Provider:** Flowing correctly (states + telecom carriers)

## Kaleyra API Details

- **IVR Number:** 8068921234
- **Agent Numbers:** 9201972062, 6264437618, 9201952685, 9201836530, 9238109518, 9201972072
- **Call Flow:** Customer → IVR (Incoming) → CallForward to agents (sequential)
- **API:** `POST https://api-voice.kaleyra.com/v1/` with `method=dial` + `api_key` + `fromdate/todate/page/limit`
- **Datetime format:** "2026-02-24 14:45:15" (IST) — parsed with +05:30 offset

## Frontend Tasks

Documented in `~/Desktop/CUSTOMER-SUCCESS/frontend-tasks.md` — Kaleyra Call Dashboard section.
Needs: `useKaleyraDashboard` hook, `useKaleyraCallList` hook, dashboard page with stat cards + charts + agent table + call log.
