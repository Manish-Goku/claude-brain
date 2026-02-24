# Notifications Module — Session Doc

**Date:** 2026-02-23
**Status:** Complete (backend + frontend wired)

## What was built

### Supabase Tables (Realtime enabled)
1. **`notifications`** — main store (type, title, message, priority, status, to_user, approval fields)
2. **`approval_requests`** — linked via `notification_id` FK (type, requested_by, status, reviewed_by)
3. **`notification_preferences`** — per-user settings (user_id+name unique, email/sms/push booleans)

### NestJS Module: `src/notifications/`
- **Service** (10 methods): create_notification, find_all_notifications, update_notification, mark_all_read, get_unread_count, find_all_approval_requests, approve_request, reject_request, get_preferences, upsert_preferences
- **Controller** (10 endpoints): all under `/notifications`
- **DTOs**: notification.dto.ts, approvalRequest.dto.ts, notificationPreference.dto.ts
- **Module exports `NotificationsService`** — injectable by other modules (e.g. ivrWebhooks can create missed call notifications)

### Key design decisions
- `requires_approval=true` on create → auto-creates `approval_requests` row linked via `notification_id`
- Approve/reject updates both `approval_requests` AND linked `notifications` table
- Preferences auto-seed 6 defaults on first GET (missed_calls, sla_breaches, escalations, assignments, approval_requests, system_updates)
- No WebSocket from backend — frontend subscribes to Supabase Realtime directly

### Endpoints
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | /notifications | Create notification |
| GET | /notifications | List with filters (type, status, priority, to_user, search, page, limit) |
| GET | /notifications/unread-count/:user_id | Badge count |
| PATCH | /notifications/mark-all-read | Bulk mark read |
| PATCH | /notifications/:id | Mark single read/actioned/dismissed |
| GET | /notifications/approval-requests | List approvals |
| POST | /notifications/approval-requests/:id/approve | Approve |
| POST | /notifications/approval-requests/:id/reject | Reject |
| GET | /notifications/preferences/:user_id | Get prefs (auto-seeds) |
| PUT | /notifications/preferences/:user_id | Upsert prefs |

## Frontend wiring (2026-02-23)

### Hook: `src/hooks/useNotifications.ts`
Pattern follows `useIVRSidebarCounts` + `useLiveChat`:
- **State**: `notifications[]`, `approvalRequests[]`, `unreadCount`, `loading`
- **Reads** (Supabase client direct): `fetchNotifications()`, `fetchApprovalRequests()`
- **Mutations** (backend API via fetch): `markRead(id)`, `markDismissed(id)`, `markAllRead(toUser)`, `approveRequest(id, reviewedBy, comments?)`, `rejectRequest(id, reviewedBy, comments)`
- **Computed** (useMemo): `slaCount`, `missedCallCount`, `pendingApprovals`
- **Realtime**: subscribes to `notifications` + `approval_requests` tables for `*` events → refetch on change
- Maps snake_case DB fields → camelCase frontend types (reuses `Notification`/`ApprovalRequest` interfaces from `notificationData.ts`)

### Files modified
| File | What changed |
|------|-------------|
| `src/integrations/supabase/types.ts` | Added `notifications`, `approval_requests`, `notification_preferences` table types |
| `src/hooks/useNotifications.ts` | **NEW** — full hook with fetches, mutations, realtime, computed stats |
| `src/pages/Notifications.tsx` | Replaced all mock imports (`NOTIFICATIONS`, `APPROVAL_REQUESTS`, `getUnreadCount`, `getPendingApprovals`) with hook. Wired: Refresh, Mark All Read, inline approve/reject, panel approve/reject, rejection dialog |
| `src/components/layout/AppHeader.tsx` | Bell badge → dynamic `unreadCount` (hidden when 0), dropdown → latest 3 unread notifications |
| `src/components/layout/AppSidebar.tsx` | Notifications nav badge → live `unreadCount` from hook |
| `src/pages/Settings.tsx` | Notification preferences fetched from `GET /notifications/preferences/manish` on mount, toggles persist via debounced `PUT` call (500ms) |
