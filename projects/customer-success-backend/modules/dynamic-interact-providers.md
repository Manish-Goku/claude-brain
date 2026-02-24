# Dynamic Interact Providers — Multi-Account WhatsApp Channels

## Date: 2026-02-24

## What was done
Same dynamic provider pattern as IVR — applied to Live Chat (Interact/WhatsApp). New providers can be added from the UI without code changes. Each provider gets its own webhook URL + API key, and conversations are scoped per-provider so the same customer phone via different channels creates separate conversations.

Unlike IVR providers, no `field_mapping` or `status_map` needed — all Interact accounts send the same 2 event types (`message_received`, `workflow_response_update`).

**Verified end-to-end**: Created `sales-whatsapp` provider, sent webhook → separate conversation created with `channel: 'sales-whatsapp'`. Old `/webhooks/interakt` endpoint still works (backward compat).

## Database

### `interact_providers` table
| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| slug | text UNIQUE | URL-safe ID — maps to `conversations.channel` |
| name | text | Display label ("Interakt (WhatsApp)") |
| api_key | text | Auto-generated 48-char hex, webhook auth |
| channel_phone_number | text | WhatsApp number for this channel |
| icon | text | Lucide icon name (default: 'smartphone') |
| display_order | int | Sidebar/tab ordering |
| is_active | boolean | Soft-delete toggle |

- RLS: `USING (true)` policy for service role
- Seeded 1 row: `interakt` (the existing default provider)

### `conversations` unique constraint change
- **Before**: `UNIQUE (phone_number)` — only 1 conversation per phone across all channels
- **After**: `UNIQUE (phone_number, channel)` — separate conversations per channel
- Migration: `update_conversations_unique_constraint_add_channel`

### Current providers (as of 2026-02-24)
| Slug | Name | Phone | Order |
|------|------|-------|-------|
| interakt | Interakt (WhatsApp) | 918219953132 | 1 |

## Backend Changes (customer-success-backend)

### `chatWebhook.controller.ts`
Added `POST /webhooks/interact/:slug` endpoint:
- `@Param('slug')` — provider slug
- `@Headers('x-api-key')` — API key for authentication
- Fire-and-forget pattern (returns 200 immediately, processes async)
- Delegates to `chat_ingestion_service.process_interact_webhook(slug, api_key, body)`

### `chatIngestion.service.ts`
4 changes:

1. **New `process_interact_webhook()` method** — looks up provider in `interact_providers` by slug + is_active, validates API key, delegates to `process_webhook(payload, slug)`

2. **`process_webhook()` now accepts `channel` param** — defaults to `'interakt'` for backward compat with old `/webhooks/interakt` endpoint. Threads channel to both `process_message_received()` and `process_workflow_response()`.

3. **`process_message_received()` and `process_workflow_response()` thread `channel`** — pass it through to `ingest_inbound_message({ ..., channel })`

4. **Conversation lookup scoped by channel** — changed from:
   ```typescript
   .eq('phone_number', phone_number).single()
   ```
   to:
   ```typescript
   .eq('phone_number', phone_number).eq('channel', channel).maybeSingle()
   ```
   `.single()` → `.maybeSingle()` because customer may not have a conversation on this specific channel yet.

### Import added
`UnauthorizedException` from `@nestjs/common` (for invalid API key).

## Test Results
| # | Test | Result |
|---|------|--------|
| 1 | Unknown slug `/interact/nonexistent` | 200, server logs `NotFoundException` |
| 2 | Wrong API key | 200, server logs `UnauthorizedException` |
| 3 | Valid interakt via dynamic endpoint | Conversation created, `channel: interakt` |
| 4 | Same phone via `sales-whatsapp` | **Separate** conversation, `channel: sales-whatsapp` |
| 5 | Old `/webhooks/interakt` (backward compat) | Works, defaults `channel: interakt` |

## Bug Found & Fixed During Testing
`conversations` table had `UNIQUE (phone_number)` — inserting a second conversation for same phone on different channel threw `duplicate key violation`. Fixed with migration to `UNIQUE (phone_number, channel)`.

## Files Modified
| File | Change |
|------|--------|
| `src/chatIngestion/chatWebhook.controller.ts` | Added `POST /webhooks/interact/:slug`, imported `Param`, `Headers` |
| `src/chatIngestion/chatIngestion.service.ts` | Added `process_interact_webhook()`, threaded `channel` through pipeline, scoped conversation lookup |
| Supabase migration: `create_interact_providers` | New table + RLS + seed |
| Supabase migration: `update_conversations_unique_constraint_add_channel` | `UNIQUE(phone_number)` → `UNIQUE(phone_number, channel)` |
| `~/Desktop/frontend-tasks.md` | Added "Dynamic Interact Providers" section with 8 frontend tasks |

## Frontend Tasks (documented in `~/Desktop/frontend-tasks.md`)
1. `useInteractProviders` hook — CRUD + realtime on `interact_providers`
2. Masters page: "Interact Providers" tab (simpler than IVR — no field_mapping/status_map)
3. Dynamic sidebar: Live Chat children from providers + static Netcore
4. `useChatCounts`: count per provider slug
5. `useLiveChat`: add channel filter param
6. LiveChat page: dynamic tabs from providers
7. `DepartmentSubTabs`: add `useChatChannelTabs()`
8. Supabase types: add `interact_providers` table type

## Key Design Decisions
- **No field_mapping/status_map** — unlike IVR providers, all Interact accounts share the same webhook payload format
- **Separate from old endpoint** — `/webhooks/interact/:slug` (new) vs `/webhooks/interakt` (legacy, kept for backward compat)
- **`channel` column is the pivot** — conversations, counts, sidebar tabs all key off `conversations.channel`
- **API key auth** — each provider gets auto-generated key, validated on every webhook call
