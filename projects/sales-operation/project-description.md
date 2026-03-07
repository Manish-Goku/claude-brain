# sales-operation

## Overview
Full-stack sales operations dashboard for Katyayani Organics. Separate from ko-sales-backend — different stack (Python/FastAPI vs NestJS), different purpose (operations management vs CRM/sales pipeline). Manages orders, leads, agents, reports, agronomy, SOPs.

## Stack

### Backend (`sales-operation-backend-1`, primary branch: `V.01`, working branch: `manish-master`)
- **Framework**: FastAPI >=0.95 (Python 3.13 local, **Python 3.9 on EC2 production** — NEVER use `str | None` union syntax, always use `Optional[str]`)
- **Server**: Uvicorn, port 8000
- **DB**: MongoDB `CRM-Database` (shared with CRM-Backend, Inventory-Management-Backend, utilities-go)
- **Auth**: JWT HS256 (python-jose), bcrypt (passlib), 8hr expiry
- **External**: Firebase Admin (FCM, Firestore), Slack API (OAuth+bot), Stockship proxy, Supabase PostgreSQL (attendance), boto3 (S3)
- **Entry**: `main.py` — lifespan starts MongoDB, Firebase, AssignmentSystem, DBListener
- **Deploy**: Render (free) → `sales-operation-backend.ko-tech.in`

### Frontend (`sales-operations`, primary branch: `main`, working branch: `manish-master`)
- **Framework**: Vite 5.4 + React 18.3 + TypeScript 5.8 + SWC
- **UI**: shadcn-ui (50+ Radix primitives) + Tailwind 3.4
- **State**: No Redux — localStorage + AuthContext + LeadsFilterContext + per-hook state
- **Data**: Custom hooks with in-memory cache (React Query installed but underused)
- **Realtime**: Firebase Realtime DB (5s polling), Firestore (onSnapshot chat)
- **Deploy**: `agri-sales-hub.ko-tech.in`

## Commands
```bash
# Backend
cd ~/Desktop/sales-operation/sales-operation-backend-1
pip install -r requirements.txt && uvicorn main:app --reload --port 8000

# Frontend
cd ~/Desktop/sales-operation/sales-operations
npm run dev    # port 5173, proxy /api→:8000
npm run build  # vite build → dist/
```

## Backend Modules
```
main.py                          # Entry, lifespan, CORS, route registration
auth/                            # JWT auth, signup, role assign/unassign, module access permissions
dashboard/                       # Order summary, pending, NDR, escalated, failed, tracking, invoice download
leads_operation/                 # Analytics, batches, clusters, workflows, assignment
leads_management/                # Lead request approval flow
Inventory/                       # Komal API proxy (warehouse inventory, native filters, stats counts)
agronomy/                        # Call audits (14 params, server-side search, stats API), tests (MCQ auto-eval), question bank, suggestions, skill matrix (aggregation pipeline), principal certs (S3 uploads), products, FCM notifications
sop/                             # SOP CRUD (Firebase Storage attachments, versioning)
Reports_and_Analytics/           # Manager/agent revenue, performance, attendance
coupons/                         # Coupon CRUD, rule engine (nested all/any conditions), eligibility check, apply, usage logs
stockship_proxy/                 # Proxy to beta.stockship.ko-tech.in
utilities/db_listener.py         # MongoDB change stream → auto lead assignment
utilities/assignment_logic.py    # Workflow engine, round-robin
utilities/slack/                 # Slack OAuth + messaging
utilities/fcm_utils.py           # Firebase push notifications
utilities/postgres_utils.py      # Supabase attendance
workers/lifecycle.py             # Assignment system lifecycle
```

## Frontend (54 pages, 50+ hooks)
Key pages: Dashboard, AllOrders, SupportPending, PreDispatch, NDR, RTO, Failed, OutOfStock, LeadMaster (6 subtabs), CallAudits, AgronomyTests, SkillMatrix, FloorManager, SalesHead, UserManagement, RoleManagement, ProfileTracker

## Auth
- **Roles**: sales_admin, sales_head, floor_manager, support_manager, junior_support, senior_agronomy, junior_agronomy, ba_team
- **Default signup role**: `retention` (can't login)
- **Module access**: permissions_matrix collection → frontend sidebar filtering
- **localStorage**: backend_access_token, backend_user_email, backend_user_roles, user_password (base64)
- **Hardcoded agronomy creds**: praveen.dev@katyayaniorganics.com / 12345
- **Super admin**: manishpandey@katyayaniorganics.com (all 8 roles, created directly in MongoDB)

## Key Constants
- SLA: 4 working hours (10AM-7PM)
- Unassigned agents: A0-1126 through A0-1130
- Order status: 26 codes (0-55, see STATUS_MAP in dashboard/route.py)
- Lead debounce: 10s in change stream
- Cache TTL: 5min (frontend)
- Agent poll: 5s (Firebase)

## Agronomy Module (21 endpoints)
- **Call Audits**: Dynamic scoring via `skills_library` collection (not hardcoded). Each skill stored as top-level field in `call_audit`. `CallAuditItemResponse` uses `model_config = {"extra": "allow"}` for dynamic passthrough. `_id` must be popped (not indexed) before Pydantic serialization.
  - **Audit status in call logs**: Backend `$lookup` joins `call_audit` into connected calls pipeline, returning `is_audited`, `total_score`, `max_total_score`, `percentage_score`, `audit_id` per call. Frontend shows colored badge (green ≥70%, amber ≥40%, red <40%) on audited rows.
  - **Server-side search**: Dropdown with Call ID, Lead ID, Lead Name, Agent Name options. Debounced (500ms). Backend `build_search_filter()` handles all types:
    - Call ID → regex on `calls.callId`
    - Lead ID → queries `leads.leadId` (normalizes KO-/ko- → K0-), then filters calls by matching `customer_id`s
    - Lead Name → queries `leads.firstName`, then filters calls by matching `customer_id`s
    - Agent Name → `_search_agents_by_name()` handles multi-word queries (splits words, matches each against `agents.firstname`/`lastname`), then filters calls by matching `agent_id` (email)
  - **Stats endpoint**: GET `/call-audits/stats` → returns total_calls, audited_count, pending_count, breach_count, avg_score for full date range (dashboard cards use this, not page-level data)
  - **Department filter**: Dynamic dropdown from GET `/agronomy/agent-categories` (distinct `Agent_Category` from agents collection). Auto-updates when new categories are added.
  - **Frontend**: `useCallLogs` + `useCallAuditStats` hooks. 30-day default. View Details dialog is fully dynamic (no hardcoded params).
  - **Inline audio player**: Both Complete Audit and View Details dialogs embed native `<audio controls>` for listening while scoring. `CallDetails` model includes `call_recording` URL.
  - **Score=0 fix**: Dropdown uses `call.total_score != null` (not truthiness) to distinguish audited vs unaudited calls.
  - **Call details column**: Shows lead_name (or lead_id if no lead, or N/A). Lead ID shown below in blue mono only when lead_name exists.
  - **IST timezone fix**: Appends 'Z' to UTC datetime strings from MongoDB before parsing in `formatDateToIST()`.
  - **Agent Detail Audits**: `AgentDetailDialog` audits tab shows real data from `get_call_audits_by_email()` with expandable skill breakdown per audit (clickable rows reveal skill scores grid). `CallAudit` response model includes `total_score`, `max_total_score`, `percentage_score`, `skill_scores` dict, `audit_id`.
- **Tests**: MCQ auto-eval (exact string match), tracks `given_by`/`pending_agents`. Collections: `agronomy_tests`, `agronomy_questions`, `agronomy_answer_sheets`. Frontend uses CRM API (`beta.api.crm.ko-tech.in`) via useAgronomyTests hook (13 methods).
- **Skill Matrix**: Precomputed `skill_matrix` collection (not live aggregation). On every audit create/update, `refresh_agent_skill_matrix()` recalculates that agent's avg scores and upserts. `overall_score` uses percentage formula: `(sum_of_avg_scores / sum_of_max_scores) * 100`. Reading is a simple `find()` — supports pagination (page/limit), department filter, sorting. Response time: ~0.3s (was 75s with aggregation pipeline). GET `/agronomy/skill-matrix?page=1&limit=25&department=Retention`. Frontend `useSkillMatrix(page, limit, department)` with `useCallback` + exposed `refetch`. Department filter is server-side param.
- **Skills Library**: CRUD for scoring skills. Collection: `skills_library`. Endpoints: GET/POST/PUT/DELETE `/agronomy/skills`. Active skills drive audit form, scoring, and skill matrix.
- **Suggestions**: Crop issues with photo uploads (Firebase Storage). Collection: `agronomy_suggestions`. Stages: Vegetative/Panicle/Flowering/Fruiting/Mature Harvest.
- **Principal Certificates**: B2B customer registration with S3 doc uploads. Collection: `principal_certificates`.
- **Products**: Active products from `products` collection joined with `categories`.
- **FCM**: Test notifications via `send_fcm_to_topic()`, email→topic normalization.

## Training Module
- **SOPs**: CRUD with versioning, Firebase Storage attachments. Auth: `require_operation_sales_role`. Frontend: useSOPs hook. Access: `reports_analytics.training_sops`.
- **New Joining**: UI only (7 stages: waiting→basic_training→crm_training→video_content→skill_assessment→test→ready). No backend.
- **Training Programs**: UI only (in SkillMatrix page). No backend persistence.

## Coupon System (10 endpoints)
- **Module**: `coupons/` — route.py, queries.py, evaluation.py, request_model.py, response_model.py
- **Collections**: `sales_coupons` (rules/config), `sales_coupon_usage` (per-usage tracking). Uses `sales_` prefix to avoid clash with existing CRM `coupons` collection.
- **Rule Engine**: Custom Python evaluator (~60 lines, zero deps). 12 operators, recursive `all`/`any` nesting. Conditions stored as JSON in MongoDB, evaluated at eligibility check time.
- **Endpoints**: POST `/coupons/create`, GET `/list`, GET `/stats`, GET `/usage-logs`, GET `/rule-metadata`, GET `/{id}`, PUT `/{id}` (versioned changelog), DELETE `/{id}` (soft), POST `/check-eligibility`, POST `/apply`
- **Discount types**: `percentage` (with optional `discount_upto` cap), `fixed`
- **Limits**: overall (total uses), per-customer, per-agent (monthly). -1 = unlimited.
- **Scope**: `all` | `product` (SKU list) | `category` (category names)
- **Flat filters**: allowed/blocked states, payment modes, customer categories, min order value/quantity
- **Nested conditions**: JSON rule tree for complex logic (e.g., "product X AND order >= ₹2000 AND (state=MH OR state=KA)")
- **Auto-gen codes**: `CPN-{incrementing_number}` if code not provided
- **Frontend**: `useCoupons` hook, `Coupons.tsx` (management + redemption tabs, 4 stat cards), `CreateCouponDialog.tsx` (visual condition builder with groups, AND/OR logic, variable/operator/value pickers), View Details dialog (full coupon info + rendered condition tree)
- **Auth**: `require_pbi_or_retention()` (same as agronomy)

## Data Model: Calls → Leads → Agents (OLD schema, current production)
- **calls**: `callId`, `customer_id` (= lead ID like K0-xxx), `agent_id` (= agent email), `datetime`, `outcome`, `duration`, `callRecording`, `phoneNumber`
- **leads**: `leadId` (K0-xxx), `firstName`, `contact`, `State`
- **agents**: `email`, `firstname`, `lastname`, `agentId` (A0-xxx), `Agent_Category`, `phone`, `manager_id`
- **Join chain**: `calls.customer_id` → `leads.leadId` for lead name; `calls.agent_id` → `agents.email` for agent name
- **Data overlap**: ~24K unique customer_ids in calls, ~10K leads, only ~880 overlap. Most calls have no matching lead.
- **Knowledge-base** (`~/Desktop/knowlegde-base/`) documents TARGET/NEW schema (pii_id, call_time, lead_id, leads_v2) but actual DB data still uses OLD field names. leads_v2 only has ~2000 records (migration incomplete).

## Dev Wiki
Full documentation at `~/Desktop/dev-wiki/sales-operation/` — architecture, API reference, database schemas, frontend pages, auth, integrations, lead assignment, frontend-backend mapping, agronomy module, training module, coupon system.

## Field Name Gotchas (DB)
- `State` capital S in leads, `assigned_Date` capital D
- `desposition` TYPO in calls (not disposition)
- `Agent_Category` capitals in agents
- `total_amt` in orders (not total_amount)
- `date` field for range filtering in orders (not order_date)
- `customer_id` in calls = lead ID (K0-xxx) — NOT an ObjectId
- `agent_id` in calls = agent email — NOT agentId (A0-xxx)
- `outcome` values: "Connected" or "connected" (both exist, filter with `$in`)

## Inventory Module (Komal API)
- **Data source**: Komal warehouse management API (`api.komal.ko-tech.in`) — NOT MongoDB
- **Auth**: OAuth2 password grant → Bearer token (9h cache). Credentials in `.env` (`KOMAL_BASE_URL`, `KOMAL_USERNAME`, `KOMAL_PASSWORD`, `KOMAL_DEFAULT_STORE`)
- **Komal endpoint**: `GET /inventory/{store_name}` — params: `page`, `page_size`, `status` (negative/low/high), `filters` (JSON array)
- **Native filters**: `[{"field":"category","filters":[{"operator":"contains","operator_value":null,"value":"FG"}]}]` — works for category, name, sku. Multiple filters AND-ed. No server-side caching needed.
- **Backend endpoints**: `GET /inventory/inventory` (listing), `GET /inventory/stats` (3 calls to Komal for out_of_stock/low_stock/available counts), `GET /inventory/stores` (7 warehouses)
- **Status mapping**: negative→out_of_stock, low→low_stock, high→available
- **Stores**: Gilehri Warehouse (~3931 items, default), Katyayani Organics (~1629), Gilehri B, Gilehri International, Gilehri W3, Gilehri W4, Gilehri Workstation 2
- **Frontend**: `OutOfStock.tsx` — store selector, status/category/search filters (all server-side), stats cards via `useInventoryStats`, table via `useInventory`. ~1s per request.

## Default Date Ranges
- **Call Audits**: 30 days (applied on mount via preset)
- **All Orders**: 30 days
