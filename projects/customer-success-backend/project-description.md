# Customer Success Backend

## Vision
Unified customer success portal for Katyayani Organics. Consolidates all customer queries (emails, calls, chats) from multiple platforms into one place instead of managing them separately.

### Channels
| Channel | Platform | Status |
|---------|----------|--------|
| Email | Gmail (API + Pub/Sub) | Built (GCP setup pending) |
| Chat | Interakt (WhatsApp) | Planned |
| Calls | IVR | Planned |

### Core Features
- **Omnichannel inbox** — emails, chats, calls in one view
- **AI-powered triage** — Gemini auto-summarizes and suggests team assignment (finance, support, dispatch, sales, technical, returns_refunds, general)
- **Agent assignment** — assign queries to specific support agents
- **Admin dashboard** — daily volume, query types breakdown, agent performance
- **Real-time** — WebSocket push to frontend for all channels

## Stack
- **Framework:** NestJS 11, TypeScript (strict)
- **Database:** Supabase (PostgreSQL)
- **AI:** Google Gemini 2.0 Flash (summarization + classification)
- **Port:** 3002
- **Location:** `~/Desktop/customer-success-backend/`

## Naming Conventions
- Variables & functions: `snake_case`
- Files & folders: `camelCase`
- Classes & interfaces: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`

## Current Architecture
```
Gmail Inbox → Gmail API watch() → Pub/Sub → POST /webhooks/gmail → Fetch email → Gemini AI (summarize + classify) → Store in Supabase → Push via WebSocket
```

## Key Modules (Built)
- `supabase/` — Global Supabase client (service_role key)
- `gmailIngestion/` — Gmail API wrapper, CRUD for support emails, webhook handler, cron for watch renewal, AI summarization
- `emailGateway/` — WebSocket gateway (Socket.io, namespace `/emails`)

## Key Modules (Planned)
- `dashboard/` — Admin analytics (daily volume, query types, agent stats)
- `agents/` — Support agent management, assignment logic
- `interakt/` — Interakt/WhatsApp chat ingestion
- `ivr/` — IVR call log ingestion

## Supabase
- Project URL: `https://gusenrddxwuwrclzfkur.supabase.co`
- Tables: `support_emails`, `emails` (with `summary` + `suggested_team` columns)
- Future tables: `agents`, `assignments`, `chats`, `calls`, `dashboard_stats`

## GCP
- Project ID: `ultra-glyph-488212-v1`
- APIs enabled: Gmail API, Cloud Pub/Sub API
- Service account: NOT YET CREATED (needs Pub/Sub Editor role permission)

## Pending Setup (BLOCKED — needs GCP permissions)
1. Create service account `email-ingestion` with **Pub/Sub Editor** role
2. Enable **domain-wide delegation** on the service account
3. Authorize in **Google Workspace Admin** → Security → API Controls → Domain-wide Delegation with scope `https://www.googleapis.com/auth/gmail.readonly`
4. Create Pub/Sub topic `gmail-push-notifications`
5. Grant `gmail-api-push@system.gserviceaccount.com` **Publisher** role on the topic
6. Create push subscription → `https://<domain>/webhooks/gmail`
7. Fill `GOOGLE_*` env vars in `.env`
8. Add `GEMINI_API_KEY` to `.env`
9. Test end-to-end: add support email → send test email → verify ingestion + AI + WebSocket

## API Endpoints
| Method | Path | Description |
|--------|------|-------------|
| POST | `/support-emails` | Add email to monitor |
| GET | `/support-emails` | List monitored emails |
| GET | `/support-emails/:id` | Get single |
| PATCH | `/support-emails/:id` | Update (toggle active, name) |
| DELETE | `/support-emails/:id` | Remove + stop watch |
| POST | `/support-emails/:id/watch` | Manual watch start |
| POST | `/support-emails/:id/stop-watch` | Manual watch stop |
| GET | `/emails` | List emails (paginated) |
| GET | `/emails/:id` | Full email details (with AI summary + team) |
| POST | `/webhooks/gmail` | Pub/Sub push endpoint |

## Swagger
`http://localhost:3002/api/docs`
