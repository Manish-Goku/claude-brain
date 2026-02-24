# Dynamic IVR Providers — Auto-register via Webhook

## Date: 2026-02-23

## What was done
Replaced all hardcoded IVR departments (general, retailer, app, kyc, international) spread across ~15 files with a fully dynamic system. Admins register IVR providers in Masters page → get webhook URL + API key → frontend auto-generates sidebar, tabs, badge counts.

Design choices: generic webhook with configurable field mapping, 1 provider = 1 department, admin UI in Masters page.

**Verified end-to-end**: Created "Manish IVR" provider via DB, sent webhook call → appeared in sidebar, department page, and Masters tab with zero code changes.

**Critical fix**: Had to manually add `ivr_providers` table definition to `src/integrations/supabase/types.ts` — the Supabase typed client silently rejects queries to tables not in the types file.

## Database

### `ivr_providers` table
| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| slug | text UNIQUE | Join key — webhook writes `department = slug` into `ivr_calls` |
| name | text | Display label ("General IVR") |
| api_key | text UNIQUE | Webhook auth |
| field_mapping | jsonb | Dot-notation paths: `{"mobile_number":"data.caller", ...}` |
| status_map | jsonb | Provider status → canonical: `{"ANSWERED":"active","NO_ANSWER":"missed"}` |
| icon | text | Lucide icon name (default: 'Phone') |
| did_numbers | text[] | DID numbers for this provider |
| display_order | int | Sidebar/tab ordering |
| is_active | boolean | Soft-delete toggle |

- RLS: authenticated read, service_role full access
- Realtime publication enabled
- Seeded 5 rows for existing departments + 1 test (manish)

### Current providers (as of 2026-02-23)
| Slug | Name | Icon | Order |
|------|------|------|-------|
| general | General IVR | Phone | 1 |
| retailer | Retailer IVR | Users | 2 |
| app | App IVR | Smartphone | 3 |
| kyc | KYC IVR | FileCheck | 4 |
| international | International IVR | Globe | 5 |
| manish | Manish IVR | Headphones | 6 |

## Backend (customer-success-backend)

### Webhook Module — `src/ivrWebhooks/`
| File | Purpose |
|------|---------|
| `ivrWebhooks.module.ts` | NestJS module registration |
| `ivrWebhooks.controller.ts` | `POST /webhooks/ivr/:slug` — fire-and-forget (returns 200 immediately) |
| `ivrWebhooks.service.ts` | Lookup provider, validate API key, apply field_mapping + status_map, upsert `ivr_calls` |

- `resolve_path()` — dot-notation traversal for field mapping (e.g. `data.caller.phone`)
- Auto-generates `call_id` as `${SLUG}-${timestamp}` if not mapped
- Registered in `app.module.ts`

### Webhook test
```bash
# Valid call
curl -X POST \
  -H "x-api-key: {api_key}" \
  -H "Content-Type: application/json" \
  -d '{"data":{"caller":"9876543210","callerName":"Test Farmer","status":"WAITING","did":"1800123456","callId":"TEST-001"}}' \
  http://localhost:3002/webhooks/ivr/{slug}

# Invalid key → returns 200 but no row inserted (fire-and-forget, error logged server-side)
```

## Frontend (katyayani-customer-success)

### New Files
| File | Purpose |
|------|---------|
| `src/hooks/useIVRProviders.ts` | CRUD + realtime subscription on `ivr_providers`. Returns `providers, getLabel, labelMap, createProvider, updateProvider, deleteProvider` |
| `src/hooks/useCallCategories.ts` | Fetches `call_categories` table by department. Replaces hardcoded `DEPARTMENT_CATEGORIES` |
| `src/lib/iconMap.ts` | Maps ~20 lucide icon name strings → React components. `getLucideIcon(name)` with Phone fallback |
| `src/components/masters/IVRProvidersTab.tsx` | Full CRUD UI: table, add/edit dialog, field mapping editor, status map editor, webhook URL display, API key management |

### Key Refactors
1. **`useIVRSidebarCounts.ts`** — Changed from fixed `{ general: number, retailer: number, ... }` to dynamic `{ [slug: string]: number }` for both `live` and `hangup`
2. **`AppSidebar.tsx`** — Builds IVR Live/Hangup children dynamically from `useIVRProviders()`. Badge via `ivrCounts.live[p.slug]`
3. **`DepartmentSubTabs.tsx`** — New `useIVRDepartmentTabs()` hook builds tabs from providers. Static `IVR_LIVE_DEPARTMENTS`/`IVR_HANGUP_DEPARTMENTS` kept as empty fallbacks
4. **`ivrData.ts`** — `IVRDepartment` changed from union type to `string` alias. `DEPARTMENT_LABELS`/`DEPARTMENT_CATEGORIES` kept but unused
5. **Department pages** (`IVRLiveDepartment`, `IVRHangupDepartment`, `IVRConsolidated`, `CallHistory`) — replaced static label/category lookups with `useIVRProviders` and `useCallCategories` hooks
6. **IVR components** (`AgentCallScreen`, `MissedCallsSection`, `CallHistoryTab`) — `department` prop type: `IVRDepartment` → `string`
7. **Masters.tsx** — Added "IVR Providers" tab
8. **`src/integrations/supabase/types.ts`** — Added `ivr_providers` table definition (required for typed client)

### Masters UI — IVR Providers Tab
- **Table**: Name (with icon), Slug, Webhook URL (copyable), API Key (masked + copy), DID count, Status toggle, Actions
- **Add/Edit Dialog**: Name, Slug (auto-from-name), Icon picker, Display order, Active toggle, DID numbers, Field Mapping editor (key-value rows), Status Map editor (key-value rows)
- **Delete**: Soft-delete (is_active = false) with confirmation
- **Regenerate Key**: Confirmation + `crypto.randomUUID()`

## Files Modified (all ~16)
- `src/hooks/useIVRSidebarCounts.ts` (rewritten)
- `src/hooks/useIVRCalls.ts` (type changes)
- `src/components/layout/AppSidebar.tsx` (dynamic children)
- `src/components/shared/DepartmentSubTabs.tsx` (dynamic tabs hook)
- `src/components/shared/index.ts` (new export)
- `src/data/ivrData.ts` (type relaxation)
- `src/pages/IVRLive.tsx`, `IVRHangup.tsx` (hook switch)
- `src/pages/IVRLiveDepartment.tsx`, `IVRHangupDepartment.tsx` (hooks for labels/categories)
- `src/pages/IVRConsolidated.tsx`, `CallHistory.tsx` (dynamic labels)
- `src/components/ivr/AgentCallScreen.tsx`, `MissedCallsSection.tsx`, `CallHistoryTab.tsx` (type changes)
- `src/components/masters/index.ts`, `src/pages/Masters.tsx` (new tab)
- `src/data/mastersData.ts` (compat comment)
- `src/integrations/supabase/types.ts` (added ivr_providers table)
- Backend: `src/app.module.ts` (module registration)

## Gotcha
- **Supabase types**: New tables MUST be added to `src/integrations/supabase/types.ts` manually. The typed client silently returns empty results for unknown tables — no error thrown.

## Bug Fix: Hangup Department Pages Empty (2026-02-23)
`IVRHangupDepartment.tsx` had 3 bugs causing calls to not display despite correct sidebar counts:
1. **viewMode default `'my'`** with hardcoded `agentName='Rishi Kumar'` — filtered out all real calls. Fixed: default to `'all'`
2. **activeTab default `'my-assigned'`** — invalid in "All" mode (tabs are `pending`, `assigned`, etc.). Fixed: default to `'pending'`, reset tab on viewMode switch
3. **pendingCalls filter gap** — `!c.agentName && (c.attempts === 0 || c.attempts === undefined)` excluded unassigned calls with 1-2 attempts (e.g. CALL-019: app dept, attempts=2, no agent). Fixed: `!c.agentName && c.resolution !== 'solved' && c.resolution !== 'not_resolved'`

**Lesson**: Category filters on lines 74-79 must be exhaustive — every call in `filteredQueue` must land in at least one tab.

## Remaining / Future
- `Settings.tsx` still imports `DEPARTMENT_OPTIONS` from `mastersData.ts` — could be dynamified
- `mockCustomer`, `mockOrders` etc. in IVRLive.tsx remain mock (no customers/orders tables yet)
