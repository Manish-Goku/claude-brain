# ko-sales-backend

## Overview
NestJS 10 + TypeScript sales CRM backend for Katyayani Organics.
**Separate project** from CRM-Backend — different stack, different DB, different purpose.

## Stack
- **Framework**: NestJS 10 (not Express — decorators, DI, guards, pipes)
- **Language**: TypeScript (strict, no CJS migration needed)
- **DB**: MongoDB — `ko-sales-db` (separate from CRM-Database used by other projects)
- **ODM**: Mongoose 7 via `@nestjs/mongoose`
- **Auth**: JWT (access 7d, refresh 10d cookie), global `JwtAuthGuard`
- **Validation**: `class-validator` + `class-transformer` via global `ValidationPipe`
- **Docs**: Swagger at `/api/docs`
- **Storage**: Firebase Admin (document images)
- **Realtime**: Supabase (task sync, graceful degradation)
- **Queue**: BullMQ via `@nestjs/bullmq` (Redis-backed job queue for cohort assignment)
- **Cron**: `@nestjs/schedule`
- **Port**: 3000 | **API prefix**: `/api/v1/`

## Commands
```bash
cd ~/Desktop/ko-sales-backend
npm run start:dev        # dev with watch
npm run build            # nest build
npm run start:prod       # node dist/main
npm run lint             # eslint --fix
npm run test             # jest
npm run test:e2e         # e2e with supertest
```

## Project Structure
```
src/
├── main.ts              # Bootstrap, Swagger, global pipes/filters, CORS
├── app.module.ts        # Root module — imports all feature + common modules
├── config/              # app.config, database.config, redis.config, env.validation
├── common/
│   ├── counter/         # CounterModule — auto-increment IDs for all entities
│   ├── activity/        # ActivityLogModule — field change audit trail
│   ├── firebase/        # FirebaseStorageModule — document image uploads
│   ├── filters/         # HttpExceptionFilter — global error format
│   ├── decorators/      # @CurrentUser, @Public, @Roles
│   ├── dto/             # shared DTOs
│   └── utils/           # sla.util.ts, roles.util.ts
└── modules/
    ├── auth/            # JWT auth, OTP flow, ghost token, refresh
    ├── agents/          # Agent CRUD, round-robin, capacity-weighted assignment
    ├── pii/             # Privacy layer — phone, email, address, GST, documents
    ├── leads/           # Lead CRUD + LeadWorkflowService + InstructionProcessor
    ├── contacts/        # Contact CRUD, 3-tier visibility
    ├── deals/           # Deal CRUD + embedded quotations + DealWorkflowService
    ├── notes/           # Standalone notes (linked via ObjectId[] on entities)
    ├── pipelines/       # Workflow configs + LP/CP/DP pipeline tracking
    ├── tasks/           # Task CRUD + TaskWorkflowService + SlaEngine + Supabase + Cron
    ├── signals/         # Inbound signals (whatsapp/call/email/system) with SLA
    ├── dashboard/       # KPI stats, pipeline funnel, pending tasks, SLA breaches
    ├── lead-activity/   # Call activity log for leads
    ├── cohorts/         # Rule-based segmentation + bulk assignment (BullMQ)
    │   ├── services/cohort-assignment.service.ts   # capacity-weighted distribution
    │   └── processors/cohort-assignment.processor.ts  # BullMQ worker
    ├── bulks/           # BulkV2 CRUD — product catalog management (7 endpoints)
    ├── associate/       # B2B catalog visibility, rule engine, pricing, packaging requests
    │   ├── services/visibility-rule-engine.service.ts  # json-rules-engine + field-priority scoring
    │   ├── services/pricing-rule-engine.service.ts     # pricing tier evaluation
    │   └── schemas/packaging-request.schema.ts         # custom packaging with per-SKU approval
    ├── associates/      # Partner onboarding, CRUD, admin management, profile
    │   ├── associates.service.ts          # full CRUD, search, filtering, pagination
    │   ├── admin-associates.controller.ts # admin management, stats, approval
    │   ├── profile.controller.ts          # mobile profile endpoint
    │   └── services/associate-aggregate.service.ts # merges Associate+PII+Address
    ├── associate-events/ # Event stream (RFQ, login, cart, etc.) — powers APS
    ├── associate-features/ # APS scoring engine (priority-based sales routing)
    │   ├── aps-scoring.service.ts    # intent 40%, engagement 25%, timing 20%, cost 15%
    │   └── window-recovery.job.ts    # periodic recalculation
    ├── onboarding-config/  # Dynamic form schema with multilingual support (en/hi)
    ├── onboarding-progress/ # Step-by-step onboarding journey tracking
    ├── agreements/      # Partner T&Cs and contract documents
    ├── products/        # Product catalog (SKU, filters, featured, new)
    ├── cart/            # Shopping cart management
    ├── schemes/         # Promotional campaign rule engine
    │   ├── services/rule-engine.service.ts        # core evaluation pipeline
    │   ├── services/targeting-matcher.service.ts   # user/product matching
    │   ├── services/condition-evaluator.service.ts # AND/OR condition logic
    │   └── services/action-executor.service.ts     # benefit calculation
    ├── tiers/           # Partner tier system (bronze/silver/gold/diamond)
    ├── wallet/          # Coin wallet + transaction ledger
    ├── payment/         # Juspay/HDFC payment gateway (service-only, no controller)
    ├── tickets/         # Support tickets with file uploads + comments
    └── seed/            # Sample data seeding (POST /seed)
```

## Coding Rules (CLAUDE.md enforced)
- **All names**: `lower_snake_case` — variables, functions, classes, constants
- **Files max ~150 lines** preferred
- **Never mix responsibilities**: controller=HTTP only, service=logic+DB, DTO=validation
- **Always use NestJS exceptions**: NotFoundException, BadRequestException, etc.
- **No raw request usage** — always DTOs
- **Async/await only**
- **Swagger decorators required**: `@ApiTags`, `@ApiOperation`, `@ApiProperty`

## Auth System
- JWT payload: `{ id, user_role }`
- Strategy enriches to: `{ id, agentId, email, katyayani_id, slack_id, user_role, role_name, agent_category, is_super_admin }`
- `JwtAuthGuard` is global (APP_GUARD) — all routes protected by default
- Use `@Public()` decorator to exempt a route
- Master password in env for dev bypass
- Ghost token endpoint: generate token for any agent by `katyayani_id` (admin only)
- Refresh token stored in DB + httpOnly cookie

## ID Patterns
| Entity | Pattern | Example |
|--------|---------|---------|
| Agent | A0-{1000+N} | A0-1001 |
| PII | PII-N | PII-1 |
| Lead | K0-1XXXXX | K0-100001 |
| Contact | C-N | C-1 |
| Deal | D-N | D-1 |
| Quotation | Q-N | Q-1 |
| LeadPipeline | LP-N | LP-1 |
| ContactPipeline | CP-N | CP-1 |
| DealPipeline | DP-N | DP-1 |
| PipelineWorkflow | PW-N | PW-1 |
| Task | T-N | T-1 |
| TaskWorkflow | TW-N | TW-1 |
| Signal | SIG-XXXXX | SIG-00001 |
| CohortAssignmentJob | CAJ-N | CAJ-1 |
| Associate | KAS-XXXXXX | KAS-000001 |
| FavouriteProduct | associate_id+sku (compound unique) | — |

## Role Codes
| Code | Role |
|------|------|
| USR-1000 | Super Admin (max 4) |
| USR-1001 | Admin |
| USR-1002 | Default (sales agent) |
| USR-1003 | Team Manager |
| USR-1004 | Floor Manager |
| USR-1005 | Agro |
| USR-1019 | Floor Manager (FM role, added by mayank-dev) |

Admin roles (full data visibility): USR-1000, USR-1001, USR-1004, USR-1005
`is_admin_role()` in `src/common/utils/roles.util.ts`

## Key Schemas (actual field names)

### Lead (`leads` collection)
`lead_id`, `pii_id`, `lead_owner{agent_id, agent_ref}`, `first_name`, `last_name`, `state`, `pin_code`, `source`, `call_status`, `call_count{answered,missed_call,not_connected,switched_off}`, `disposition`, `disposition_reason`, `follow_up_date`, `status`, `lqs`, `rating`, `pipeline_id`, `updated_data[]`, `notes[]`, `tags[]`, `query[]`

Status enum: `active, dormant, contact_created, order_placed, lost, unqualified, not_performed, disable, blacklisted`
Disposition enum: `follow_up, push_to_advisory, order_placed, lead_closed, prospect, unqualified, reassign, dormant, lost, null`

### Contact (`contacts` collection)
`contact_id`, `pii_id`, `profile`, `profile_details`, `farm_details`, `status`, `call_status`, `disposition`, `agent_category`, `pipeline_id`, `document_status{adhar,pan,pc}`, `prospect_value`, `fps`, `rps`, `notes[]`, `tags[]`, `updated_data[]`

### Deal (`deals` collection)
`deal_id`, `pii_id`, `contact_id`, `lead_id`, `agent_category`, `agent_id`, `pipeline_id`, `status(open/won/lost/closed/order_created)`, `probability`, `validity_days`, `estimated_closure_date`, `actual_closure_date`, `quotation[]`, `notes[]`, `updated_data[]`

Quotation embedded: `quotation_id`, `expiry_date`, `line_item[]`, `status(draft/shared/rejected/accepted/expired)`, `payment_type`, `total_amount`, `prepaid_amount`, `cod_amount`, `channel`

LineItem embedded: `item`, `qty`, `case_qty`, `single_item_price`, `sku`, `gst`, `total_amount`, `item_type`, `requested_weight`(Number), `packaging_size`(Number), `uom`(String: kg/l/pc), `packaging_type`(String: hdpe bottle/pouch)

### PII (`piis` collection)
`pii_id`, `phone_number[]` (unique index), `email`, `country_code`, `addresses[{label,line1,line2,city,state,pincode,country}]`, `gst_numbers[{gst_number,business_name,status,registration_date}]`, `documents{pan:{id,image}, adhar:{id,image}, pc:{id,image}}`

### Signal (`signals` collection)
`signal_id`, `pii_id`, `entity_id`, `module(lead/contact/deal/other)`, `signal_type(whatsapp/call/email/system)`, `content`, `status(pending/resolved)`, `sla_deadline`, `resolved_by`, `resolved_at`, `created_by`

### Task (`tasks` collection)
`task_id`, `task_name`, `related_to_module`, `related_to_id`, `agent_category`, `agent_id`, `task_type(pipeline/manual/follow_up/other)`, `priority(low/medium/high/urgent)`, `status(pending/in_progress/completed/overdue/cancelled)`, `sla`, `sla_deadline`, `sla_started_at`, `sla_status(on_track/at_risk/breached/completed/not_applicable)`, `sla_breach_duration`, `workflow_id`, `pipeline_id`, `step_index`, `note_ids[]`, `pii_id`, `assigned_by`, `completed_by`, `completed_at`

### Agent (`agents` collection)
`agentId`, `firstname`, `lastname`, `email`, `katyayani_id`, `user_role`, `Agent_Category`, `AssignedState[]`, `daily_capacity(default:50)`, `password(select:false)`, `status`, `slack_id`, `device_token`, `is_active`, `refreshToken`

### CohortAssignmentJob (`cohort_assignment_jobs` collection)
`job_id(CAJ-N)`, `cohort_id`, `agent_ids[]`, `status(queued/processing/completed/failed)`, `total_leads`, `assigned_count`, `failed_count`, `progress_percent`, `failures[{lead_id,pii_id,error}]`, `error_message`, `initiated_by`, `started_at`, `completed_at`

### PipelineWorkflow (`pipelines_workflow` collection)
`workflow_id`, `pipeline_type(lead/contact/deal)`, `agent_category`, `stages[{name,index,description,timeline,taskflow_id,required_fields[]}]`, `is_active`
Unique index: `{pipeline_type, agent_category}`

### LeadPipeline (`lead_pipelines` collection)
`pipeline_id`, `workflow_id`, `pii_id`, `lead_id`, `agent_id`, `agent_category`, `current_stage`, `stage_history[{stage,entered_at,exited_at,moved_by}]`, `status(active/closed/completed)`, `closed_reason(owner_changed/lead_lost/completed/null)`

### BulkV2 (`bulk_v2` collection — in KO-Database)
`sku` (String, NOT bulk_sku), `trade_name`, `technical_name`, `category` (String, NOT ObjectId), `sub_category` (String), `channels{website,distributor,retailer}`, `description{english,hindi}`, `gst_rate` (String), `hsn_code`, `bulk_min_order_qty` (String), `dist_min_order_qty` (String), `technicals[{technical_sku,technical_name,value,unit}]`, `crops[]`, `diseases[]`, `dosage[]`, `features[]`, `applications[]`, `bulk_type`, `uom`, `sku_type`

### PackagingRequest (`packaging_requests` collection)
`pii_id`, `status(pending/partially_approved/approved/rejected)`, `items[{sku,packaging_size,price,moq,status(pending/approved/rejected),rejection_reason,reviewed_by}]`, `remarks`, `created_by`, `updated_by`, `created_at`, `updated_at`
Parent status auto-derives from item statuses.

### AssociateRule (`associate_partner_rules` collection)
`rule_name`(unique), `type(category/bulk/moq/cluster/recommended/popular)`, `conditions`(json-rules-engine), `event`(Mixed), `categories[]`, `sku`, `bulks[]`, `pricing[{moq,price}]`, `packaging[{type,size}]`(bulk rules only), `cluster_name`, `is_active`, `considered_as_log`, `version`, `priority`, `comment`

## Architecture Decisions
1. **PII = privacy layer** — all sensitive data (phone, email, docs) in PII. Lead/Contact reference by `pii_id` string (NOT ObjectId → no `.populate()`, manual joins)
2. **Thin service + workflow service** pattern — leads/deals have a thin DB layer service + separate orchestration service
3. **forwardRef** used for circular DI between LeadsService ↔ LeadWorkflowService
4. **Pipelines = forward-only** — stage transitions validated: target.index > current.index
5. **Contact takes priority** over lead for pipelines — if contact exists, use CP-*; if not, LP-*
6. **Quotations embedded** in Deal schema (not separate collection)
7. **Tasks dual-stored** — MongoDB (source of truth) + Supabase (frontend realtime, optional)
8. **SLA Engine** — business hours 10AM-7PM Mon-Sat; at_risk at 20% remaining
9. **Cron**: `FollowUpCronService` runs every 30min → marks overdue follow_up tasks
10. **Document_status** on Contact is auto-synced from PII.documents mutations
11. **ActivityLogService** tracks field changes across all modules
12. **Firebase** stores document images; `FirebaseStorageModule` imported globally
13. **BullMQ** for async cohort assignment — `@nestjs/bullmq`, Redis-backed, in-process worker
14. **Capacity-weighted distribution** — agents get leads proportional to `daily_capacity`, via `get_agent_capacity()` (swappable logic)

## Module Interconnections
```
Signals ──► Lead/Contact/Deal (via entity_id)
PII ──► Lead, Contact, Deal (pii_id string ref)
Lead ──► Contact (profile field triggers creation)
Contact ──► Deal (deal creation auto-closes contact pipeline)
Pipelines ──► Tasks (stage creation/transition triggers task workflow)
Tasks ──► Notes (complete with note creates linked Note + task_id)
Dashboard ──► Lead, Deal, Task, Signal, Contact (aggregation only)
Cohorts ──► Leads (bulk assignment via BullMQ, capacity-weighted)
Associate (old) ──► PII, BulkV2, PackagingRequests
Associates (new) ──► PII, Counter, Contacts (enriched via AggregateService)
AssociateEvents ──► AssociateFeatures (APS engine, event-driven scoring)
OnboardingConfig ──► OnboardingProgress (schema reference)
Schemes ──► Associates (targeting by tier/state/segment)
Wallet ◄── Payment (Juspay top-up funding)
Tickets ──► Associates (user lookup)
Tiers ──► Associates (tier progression)
```

## Env Variables
```
PORT=3000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/ko-sales-db
API_PREFIX=api
API_VERSION=v1
JWT_SECRET=
JWT_REFRESH_SECRET=
MASTER_PASSWORD=      # bypass password for dev
SPECIAL_EMAIL=        # special login email
SPECIAL_PASSWORDS=    # comma-separated
SMTP_EMAIL=           # OTP email
SMTP_PASSWORD=
SUPABASE_URL=         # optional
SUPABASE_KEY=         # optional
REDIS_HOST=127.0.0.1  # BullMQ
REDIS_PORT=6379
```

## Customer Success Module (merged 2026-02-28)

Entire customer-success-backend merged as an isolated module under `src/modules/customer-success/`.
169 files, 26 sub-modules, 160+ endpoints — all under `/api/v1/cs/*` routes.

### Key Adaptations
- **Routes**: All controllers prefixed with `cs/` (e.g. `@Controller('cs/agents')`)
- **Swagger**: All `@ApiTags` prefixed with `cs/` (e.g. `@ApiTags('cs/tickets')`)
- **Auth**: All controllers marked `@Public()` — bypasses ko-sales JWT guard
- **Supabase**: `SupabaseService` renamed to `CsSupabaseService` to avoid DI conflict
- **Env vars**: `CS_SUPABASE_URL` + `CS_SUPABASE_SERVICE_ROLE_KEY` (separate Supabase project: `gusenrddxwuwrclzfkur`)
- **Import extensions**: Stripped `.js` extensions (ko-sales uses CommonJS, CS used NodeNext)
- **Wrapper**: `CustomerSuccessModule` imports all 26 sub-modules, registered in `app.module.ts`

### CS Sub-Modules (under src/modules/customer-success/)
| Module | Route Prefix | Purpose |
|--------|-------------|---------|
| supabase | — | CsSupabaseService (Supabase client) |
| gmailIngestion | cs/support-emails, cs/emails | IMAP email polling + AI summarization |
| emailGateway | — | WebSocket for real-time email updates |
| chatIngestion | cs/conversations, cs/messages, cs/webhooks | WhatsApp (Interakt/Netcore) |
| chatTemplates | cs/chat-templates | Canned responses |
| adminDashboard | cs/admin/dashboard | Analytics dashboard |
| agents | cs/agents | Agent management + role hierarchy |
| roles | cs/roles | Role and module access |
| notifications | cs/notifications | Notifications + approval requests |
| queryAssignments | cs/query-assignments | Query dispatch to agents |
| ivrWebhooks | cs/webhooks/ivr | IVR call data ingestion |
| kaleyraVoice | cs/* | Kaleyra voice API + CDR webhook |
| slaConfig | cs/sla-config | SLA configuration |
| tickets | cs/tickets, cs/ticket-* | Ticket system (CRUD, categories, templates, routing) |
| surveys | cs/survey-* | Survey templates, campaigns, calls |
| refunds | cs/refunds | Refund management |
| audits | cs/audits | Audit logging |
| workflows | cs/workflows | Workflow automation |
| videoCallLeads | cs/video-call-leads | Video call leads |
| callCategories | cs/call-categories | Call categorization |
| youtubeLeads | cs/youtube-leads | YouTube leads |
| customerFeedback | cs/customer-feedback | Customer feedback |
| customerTimeline | cs/customer-timeline | Customer activity timeline |
| agentFeedback | cs/agent-feedback | Agent performance feedback |
| returnRca | cs/return-rca | Return root cause analysis |
| agriConsultancy | cs/agri-consultancy | Agricultural consultancy |

### CS Env Variables (added to .env)
```
CS_SUPABASE_URL=https://gusenrddxwuwrclzfkur.supabase.co
CS_SUPABASE_SERVICE_ROLE_KEY=...
ENCRYPTION_KEY=...          # AES-256 for IMAP passwords
GEMINI_API_KEY=             # Google Gemini for email AI
KALEYRA_API_KEY=...         # Kaleyra voice
KALEYRA_CALLBACK_URL=http://localhost:3000/api/v1/cs/webhooks/kaleyra/callback
KALEYRA_SYNC_INTERVAL_MINUTES=10
NETCORE_API_URL=            # Netcore WhatsApp
NETCORE_API_KEY=
```

## Last Worked

| Date | What |
|------|------|
| 2026-03-06 | BulkV2 CRUD module (7 endpoints), fixed app.module.ts duplicate imports, fixed agents.service.ts duplicate method, installed @aws-sdk/client-s3 |
| 2026-03-05 | Cohort system tested in depth (28/30 passing), 2 bugs fixed (operator validation + BSON overflow) |
| 2026-02-28 | Quotation LineItem schema extended, mayank-dev merged (13 modules), CS module merged |
| 2026-02-21 | Customer module + Cohort system + BullMQ assignment |

## Context Files in Project
- `.claude/context-memory.md` — compressed module/route/schema memory (maintained by Claude)
- `.claude/workflow-diagram.md` — full workflow diagrams for all flows
