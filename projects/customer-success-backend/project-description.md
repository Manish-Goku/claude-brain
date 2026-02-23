# Customer Success Backend

## Vision
Unified customer success portal for Katyayani Organics. Consolidates all customer queries (emails, calls, chats) from multiple platforms into one place instead of managing them separately.

### Channels
| Channel | Platform | Status |
|---------|----------|--------|
| Email | Gmail (API + Pub/Sub) | Built (GCP setup pending) |
| Chat | Interakt (WhatsApp) | Built |
| Chat | Netcore (WhatsApp) | Built |
| Calls | IVR | Frontend wired to Supabase (real-time) |

### Core Features
- **Omnichannel inbox** — emails, chats, calls in one view
- **AI-powered triage** — Gemini auto-summarizes and suggests team assignment (finance, support, dispatch, sales, technical, returns_refunds, general)
- **Agent assignment** — assign queries to specific support agents
- **Admin dashboard** — daily volume, query types breakdown, agent performance
- **Real-time** — WebSocket push + Supabase postgres_changes for all channels

## Stack
- **Backend:** NestJS 11, TypeScript (strict), port 3002
- **Frontend:** Vite + React + TypeScript + shadcn-ui, port 8080
- **Database:** Supabase (PostgreSQL), project ID: `gusenrddxwuwrclzfkur`
- **AI:** Google Gemini 2.0 Flash (summarization + classification)
- **Backend Location:** `~/Desktop/customer-success-backend/`
- **Frontend Location:** `~/Desktop/katyayani-customer-success/`

## Naming Conventions
- Variables & functions: `snake_case`
- Files & folders: `camelCase`
- Classes & interfaces: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`

## Architecture
```
Gmail    → Gmail API watch() → Pub/Sub → POST /webhooks/gmail    → AI classify → Supabase → WebSocket
Interakt → POST /webhooks/interakt                               → AI classify → Supabase → WebSocket
Netcore  → POST /webhooks/netcore                                → AI classify → Supabase → WebSocket
Frontend → Supabase JS client (anon key) + postgres_changes real-time subscriptions
```

## Key Modules (Built — Backend)
- `supabase/` — Global Supabase client (service_role key)
- `gmailIngestion/` — Gmail API wrapper, CRUD for support emails, webhook handler, cron for watch renewal, AI summarization
- `emailGateway/` — WebSocket gateway (Socket.io, namespace `/emails`)
- `chatIngestion/` — Multi-provider WhatsApp ingestion:
  - `chatWebhook.controller.ts` — POST /webhooks/interakt, POST /webhooks/netcore (fire-and-forget pattern)
  - `chatIngestion.service.ts` — Shared ingest pipeline: dedup, conversation mgmt, AI classify, WebSocket emit
  - `interakt.service.ts` — Interakt outbound API
  - `netcore.service.ts` — Netcore outbound API (Bearer auth)
  - `chatGateway.ts` — WebSocket gateway (namespace `/chats`)
  - Channel routing: `conversations.channel` determines which provider for outbound replies

## Key Modules (Built — Frontend)
- `src/hooks/useLiveChat.ts` — Conversations + messages from Supabase, send reply via backend, real-time
- `src/hooks/useChatCounts.ts` — Sidebar badge counts from conversations table, real-time
- `src/hooks/useEmails.ts` — Email listing with AI summaries
- `src/hooks/useQueryAssignments.ts` — CRUD + bulk assign
- `src/hooks/useAgentCalls.ts`, `useAgentChats.ts`, `useAgentHangups.ts`, `useAgentSLABreach.ts`, `useAgentCompleted.ts`
- `src/hooks/useIVRCalls.ts` — Unified hook for all 7 IVR pages (filters, mutations, real-time)
- `src/hooks/useIVRSidebarCounts.ts` — IVR sidebar badge counts (live/hangup per dept, SLA breach)
- `src/pages/LiveChat.tsx` — Wired to Supabase (conversations, messages, channel filtering, real-time)
- `src/pages/IVRLiveDepartment.tsx`, `IVRLive.tsx`, `IVRHangup.tsx`, `IVRHangupDepartment.tsx`, `IVRConsolidated.tsx`, `CallHistory.tsx`, `SLABreach.tsx` — All wired to Supabase via `useIVRCalls`
- `src/components/layout/AppSidebar.tsx` — Live Chat + IVR Live + IVR Hangup + SLA Breach badges from Supabase

## Supabase Tables
- `support_emails` — monitored Gmail accounts
- `emails` — with `summary` + `suggested_team` (AI-generated)
- `conversations` — `channel` (interakt|netcore), `phone_number`, `customer_name`, `status`, `assigned_agent`, `assigned_team`, `unread_count`
- `chat_messages` — `conversation_id`, `direction`, `content`, `message_type`, `media_url`, `external_message_id`, `summary`, `suggested_team`
- `chat_templates` — canned responses for chat
- `agents` — support agents with status, department, skills
- `agent_daily_stats` — daily per-agent metrics
- `ivr_calls` — IVR call records with SLA tracking
- `query_assignments` — multi-channel query assignment
- `audits`, `refund_requests`, `refund_request_products`, `refund_request_actions`
- `video_call_leads`, `customer_timeline`, `sla_configurations`
- `roles`, `modules`, `module_role_access`, `user_role_access`

## API Endpoints
| Method | Path | Description |
|--------|------|-------------|
| POST | `/support-emails` | Add email to monitor |
| GET | `/support-emails` | List monitored emails |
| PATCH | `/support-emails/:id` | Update |
| DELETE | `/support-emails/:id` | Remove + stop watch |
| GET | `/emails` | List emails (paginated) |
| GET | `/emails/:id` | Full email details |
| POST | `/webhooks/gmail` | Pub/Sub push endpoint |
| POST | `/webhooks/interakt` | Interakt WhatsApp webhook |
| POST | `/webhooks/netcore` | Netcore WhatsApp webhook |
| POST | `/conversations/:id/reply` | Send reply (routes to correct provider) |
| GET | `/chat-templates` | List templates |
| POST | `/chat-templates` | Create template |
| PATCH | `/chat-templates/:id` | Update template |
| DELETE | `/chat-templates/:id` | Delete template |
| GET | `/query-assignments` | List assignments |
| POST | `/query-assignments` | Create assignment |
| PATCH | `/query-assignments/:id` | Update assignment |
| POST | `/query-assignments/bulk-assign` | Bulk assign |

## Swagger
`http://localhost:3002/api/docs`

## GCP (BLOCKED — needs permissions)
- Project ID: `ultra-glyph-488212-v1`
- APIs enabled: Gmail API, Cloud Pub/Sub API
- Needs: service account, domain-wide delegation, Pub/Sub topic, push subscription
