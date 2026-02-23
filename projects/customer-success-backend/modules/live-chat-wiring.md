# Live Chat — Supabase Wiring & Sidebar Counts

## Date: 2026-02-23

## What was done
Wired the LiveChat page from mock/hardcoded data to real Supabase data, fixed channel sub-tab routing, and made sidebar badge counts live.

### 1. Frontend Hook — `useLiveChat.ts`
- Created `src/hooks/useLiveChat.ts`
- Fetches conversations from Supabase `conversations` table with last message per convo
- Loads messages on-demand per conversation from `chat_messages`
- Sends replies via `POST /conversations/:id/reply` to backend
- Agents fetched from `agents` table
- Real-time subscriptions on both `conversations` and `chat_messages` tables
- Key mapping: `map_source(db_channel)` → 'interakt'|'netcore', `map_status()`, `map_priority()`

### 2. LiveChat.tsx Rewiring
- Removed `mockConversations`, `activeChats` import, hardcoded `agents` array
- Added `useLiveChat` hook + `useLocation` for URL-based channel detection
- `activeChannelTab` derived from URL: `/interakt` → 'interakt', `/netcore` → 'netcore', else → 'all'
- Conversations filtered by `activeChannelTab` from `allConversations`
- Dynamic message rendering via `chatMessages.map()`
- Real `send_message()` API call replaces `console.log`
- Auto-scroll via `messages_end_ref`

### 3. Routing Fix
- `/live-chat/interakt` and `/live-chat/netcore` were falling through to `ChatChannel.tsx` (mock data) via `/:channel` route
- Added explicit routes in `App.tsx`:
  ```
  <Route path="/live-chat/interakt" element={<LiveChat />} />
  <Route path="/live-chat/netcore" element={<LiveChat />} />
  ```
- Updated `DepartmentSubTabs` to use `activeChannelTab` as `currentDepartment`

### 4. Sidebar Live Counts — `useChatCounts.ts`
- Created `src/hooks/useChatCounts.ts` — lightweight hook for sidebar badge counts
- Queries `conversations` table, groups by `channel` column
- Real-time subscription refreshes counts on conversation changes
- `AppSidebar.tsx` wired: Live Chat parent badge = total, Interakt/NetCore children = per-channel counts
- Replaced all hardcoded mock badge numbers (3, 2, 15) with live data

### 5. Supabase Types Update
- Regenerated `src/integrations/supabase/types.ts` via `mcp__supabase__generate_typescript_types`
- Now includes `channel: string` on conversations Row and `external_message_id` on chat_messages Row

## Files Created
- `src/hooks/useLiveChat.ts`
- `src/hooks/useChatCounts.ts`

## Files Modified
- `src/pages/LiveChat.tsx` — full rewiring from mocks to Supabase
- `src/App.tsx` — added explicit interakt/netcore routes
- `src/components/layout/AppSidebar.tsx` — live chat counts from Supabase
- `src/integrations/supabase/types.ts` — regenerated with channel field

## DB State
- 6 conversations: 4 interakt + 2 netcore (test data from webhook curls)
- 9 chat_messages across those conversations
