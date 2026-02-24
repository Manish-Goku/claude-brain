# Backend Modules Expansion — 2026-02-23

## What was built
8 new NestJS backend modules + 4 Supabase RPC functions, expanding the customer-success-backend from 10 to 18 modules with 95+ total endpoints.

## New Modules

### 1. Agents (`src/agents/`)
- CRUD for `agents` table (22 rows)
- Endpoints: GET/POST/PATCH/DELETE /agents, PATCH /agents/:id/status, PATCH /agents/:id/clock-in, PATCH /agents/:id/clock-out, GET /agents/:id/stats
- Daily stats from `agent_daily_stats` table (last 7 days)

### 2. SLA Config (`src/slaConfig/`)
- CRUD for `sla_configurations` table (4 tiers: vip/high/normal/low)
- Endpoints: GET /sla-config, GET/PUT /sla-config/:tier, PUT /sla-config/bulk

### 3. Roles (`src/roles/`)
- CRUD for `roles`, `modules`, `user_role_access`, `module_role_access` tables
- 11 endpoints covering role CRUD, module listing, user-role assignments, module access per role

### 4. Refunds (`src/refunds/`)
- CRUD for `refund_requests`, `refund_request_products`, `refund_request_actions`
- Endpoints: GET/POST/PATCH /refunds, POST /refunds/:id/actions, GET /refunds/stats

### 5. Audits (`src/audits/`)
- CRUD for `audits` table
- Endpoints: GET/PATCH /audits, GET /audits/stats

### 6. Video Call Leads (`src/videoCallLeads/`)
- CRUD for `video_call_leads` table
- Endpoints: GET/POST/PATCH/DELETE /video-call-leads, GET /video-call-leads/stats

### 7. Call Categories (`src/callCategories/`)
- CRUD for `call_categories` table (78 rows, tree structure)
- Endpoints: GET/POST/PATCH/DELETE /call-categories, GET /call-categories/by-department/:dept

### 8. Customer Timeline (`src/customerTimeline/`)
- CRUD for `customer_timeline` table
- Endpoints: GET /customer-timeline/:mobile, GET /customer-timeline/:mobile/summary, POST /customer-timeline

## Dashboard RPC Functions (Supabase)
4 PostgreSQL functions created for the frontend `useDashboard` hook:
- `get_dashboard_email_stats()` → email counts (today + all)
- `get_dashboard_chat_stats()` → chat conversation stats
- `get_dashboard_emails_by_team()` → email distribution by team
- `get_dashboard_daily_volume()` → daily email volume (30 days)

## Supabase Tables (26 total)
All tables already existed with data. All have types in frontend `types.ts`.

## Testing (2026-02-23)
All 27 endpoint groups verified via curl. Backend compiles clean with `tsc --noEmit`.

### Test Results (all PASS)
- Agents: 21 active, clock-in/out works, stats endpoint returns daily data
- SLA Config: 4 tiers loaded, bulk update works
- Roles: 3 roles, 15 modules, 12 user assignments
- Notifications: 21 total, create/mark-read/mark-all-read/preferences all work
- Approval flow: create with `requires_approval=true` + `approval_type` → auto-creates approval_request → approve/reject both work
- Support Emails: 1 configured, CRUD all functional
- Refunds: 8 total (4 pending, 2 SLA breached), add_action (comment) works
- Audits: 6 total (avg score 65), list/stats work
- Video Call Leads: 15 total (9 status types), stats work
- Call Categories: 78 categories, by-department returns 20 for General
- Customer Timeline: create event + summary (3 call, 1 order) work

### Frontend Integration (all verified)
All frontend pages wired to live data. See `frontend-integration-wiring.md` for details.

## Notification Admin/Agent Flow (verified)
1. Agent creates approval request → `POST /notifications` with `requires_approval=true`
2. Auto-creates linked `approval_requests` row with `notification_id` FK
3. Admin approves → `POST /notifications/approval-requests/:id/approve` → updates both tables
4. Admin rejects → `POST /notifications/approval-requests/:id/reject` → updates both tables with comments
5. Frontend sees updates via Supabase realtime subscriptions
