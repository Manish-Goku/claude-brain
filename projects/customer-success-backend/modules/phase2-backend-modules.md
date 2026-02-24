# Phase 2 Backend Modules — Built 2026-02-24

## Overview
8 new NestJS modules + 19 Supabase tables + ~67 endpoints.
All E2E tested (102/102 pass). 2 DTO bugs auto-fixed during testing.

## Migration: `create_19_backend_tables`
19 tables, all with RLS + service_role policy + indexes.
DB now has **45 tables** total (26 existing + 19 new).

## Module 1: Tickets (`src/tickets/`)
**17 files** — 4 controllers, 4 services, 7 DTOs

### Tables (6)
- `ticket_categories` — name, subcategories[], is_active
- `tickets` — ticket_id (TKT-NNNN auto-gen), customer fields, priority/channel/status/sla_status, escalation_level (0-2), tags[], assigned_to (FK→agents)
- `ticket_templates` — name, category, shortcut, subject, content, usage_count
- `ticket_routing_rules` — name, priority (1-99), conditions/action (JSONB), is_active
- `ticket_responses` — ticket_id (FK), sender_type, content, created_by
- `ticket_timeline` — ticket_id (FK), event_type, description, created_by

### Endpoints (~15)
- `POST/GET /tickets`, `GET/PATCH /tickets/:id`, `POST /tickets/:id/escalate|resolve|responses`, `GET /tickets/stats`
- CRUD `/ticket-categories`
- CRUD `/ticket-templates` + `POST /:id/increment-usage`
- CRUD `/ticket-routing-rules` + `POST /:id/toggle`

### Key Logic
- `generate_ticket_id()` — queries max ticket_id → TKT-NNNN sequential
- Escalate: increments escalation_level, sets status=escalated, adds timeline event
- Resolve: sets status=resolved + resolved_at timestamp
- find_one: returns ticket + responses + timeline in single response

---

## Module 2: Surveys (`src/surveys/`)
**11 files** — 3 controllers, 3 services, 4 DTOs

### Tables (5)
- `survey_templates` — name, description, category (nps/csat/product_feedback/delivery_feedback/custom), is_active, created_by
- `survey_questions` — template_id (FK), question_text, question_type (rating/nps/single_choice/multiple_choice/text/yes_no), options[], is_required, is_skippable, sort_order
- `survey_campaigns` — name, template_id (FK), status (draft/scheduled/active/paused/completed), target_segment, target_count, assigned_agents[], stats (JSONB)
- `survey_calls` — campaign_id (FK), customer_name/phone, order_id, status (pending/completed/not_answered/call_later/refused), attempts, nps_score, responses (JSONB)
- `survey_responses` — call_id (FK), question_id (FK), answer

### Endpoints (~15)
- CRUD `/survey-templates` + toggle + duplicate, question CRUD nested under templates
- CRUD `/survey-campaigns` + `/activate` `/pause` `/complete`
- `GET /survey-calls/campaign/:id`, `POST /:id/complete|not-answered|schedule`, `GET /analytics/:campaignId`

### Key Logic
- Duplicate: copies template + all questions with new UUIDs
- Analytics: NPS distribution (promoters 9-10, passives 7-8, detractors 0-6), completion rate, avg_nps
- Campaign status: separate activate/pause/complete endpoints (not generic toggle)

---

## Module 3: Agent Feedback (`src/agentFeedback/`)
**5 files** — 1 controller, 1 service, 2 DTOs

### Table
- `agent_feedback` — agent_id (FK→agents), agent_name, feedback_type, rating (1-5), sentiment, reference_id/type, customer fields, category, subcategory, comment, action_taken, status (pending/reviewed/actioned), reviewed_by/at

### Endpoints (5)
- `POST/GET /agent-feedback`, `PATCH /:id`, `GET /stats`, `GET /agent-summary`

### Key Logic
- Stats: total, avg_rating, by_sentiment, by_type distributions
- Agent summary: grouped by agent_id with count/avg_rating

---

## Module 4: Workflows (`src/workflows/`)
**7 files** — 1 controller, 1 service, 4 DTOs

### Tables (2)
- `workflows` — name, description, trigger_type/label/conditions/config, actions (JSONB), is_active, created_by/name, execution_count, last_executed_at
- `workflow_execution_logs` — workflow_id (FK), workflow_name, triggered_at/by, status (success/failed/partial/running), duration_ms, trigger_data, action_results (JSONB), error

### Endpoints (~10)
- CRUD `/workflows` + toggle + duplicate + execute
- `GET /workflows/logs` (all logs — static route before /:id)
- `GET /workflows/:id/logs` (per workflow)

### Key Logic
- Execute: creates log with status=running → simulates success with duration_ms
- Duplicate: clones with "(Copy)" suffix, is_active=false
- Static route `/logs` placed BEFORE dynamic `/:id` in controller

---

## Module 5: YouTube Leads (`src/youtubeLeads/`)
**6 files** — 1 controller, 1 service, 3 DTOs

### Table
- `youtube_leads` — lead_id (YT-NNNN auto-gen), source, customer fields, state/district, query, category, status, priority, assigned_to, attempts, notes (JSONB[]), is_qualified_lead, crm_transferred/reference_id

### Endpoints (6)
- `POST/GET /youtube-leads`, `PATCH /:id`, `POST /:id/action`, `POST /bulk-assign`, `GET /stats`

### Key Logic
- `generate_lead_id()` → YT-NNNN sequential
- Action: appends to notes JSONB array `{action, note, performed_by, timestamp}`, increments attempts
- Bulk assign: updates assigned_to for array of lead_ids

---

## Module 6: Customer Feedback (`src/customerFeedback/`)
**4 files** — 1 controller, 1 service, 1 DTO

### Table
- `customer_feedback` — problem, order_id, solution, advice_for_future, category (packaging/delivery/product/service/other), priority, status, created_by/name, team_lead/name, impact_count

### Endpoints (6)
- CRUD `/customer-feedback` + `GET /stats` + `GET /analytics`

### Key Logic
- Stats: by_category, by_status, by_priority distributions
- Analytics: weekly_trend (last 4 weeks), high_impact (impact_count > threshold)

---

## Module 7: Return RCA (`src/returnRca/`)
**4 files** — 1 controller, 1 service, 1 DTO

### Table
- `return_rca_records` — order_id, customer fields, order_value, return_reason, fault_category (sales/customer/delivery/quality), status (pending/classified/resolved), classified_at/by, notes

### Endpoints (4)
- `POST/GET /return-rca`, `PATCH /:id/classify`, `GET /stats`

### Key Logic
- Classify: sets fault_category + classified_by + classified_at (auto timestamp) + status="classified"
- Stats: by_status + by_fault distributions, total_value sum

---

## Module 8: Agri Consultancy (`src/agriConsultancy/`)
**6 files** — 1 controller, 1 service, 3 DTOs

### Tables (2)
- `agri_consultations` — farmer_name/phone, crop_type, issue, status (scheduled/in_progress/completed/cancelled), scheduled_at, agronomist, duration_seconds, notes
- `agri_prescriptions` — consultation_id (FK), products (JSONB[]), instructions

### Endpoints (7)
- `POST/GET /agri-consultancy`, `PATCH /:id`, `PATCH /:id/start`, `PATCH /:id/complete`
- `POST /agri-consultancy/prescriptions`, `GET /prescriptions/:consultationId`

### Key Logic
- Start: sets status=in_progress
- Complete: sets status=completed + duration_seconds + notes
- Prescriptions: products is JSONB array of {name, quantity, dosage}

---

## Bugs Fixed During Testing
1. `ticketRoutingRules.service.ts` — DTO classes missing class-validator decorators (added @IsString, @IsOptional, @IsNumber, etc.)
2. `youtubeLeads/dto/actionYoutubeLead.dto.ts` — `BulkAssignDto.lead_ids` missing `@IsArray()` + `@IsUUID()` decorators

## Registration
- All 8 modules registered in `app.module.ts` (total 26 modules)
- 14 Swagger tags added to `main.ts`
- Frontend `types.ts` already had all 45 tables
- `tsc --noEmit` passes clean
