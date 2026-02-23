# Order Tracking System

## Overview
Cron-based tracking system that polls courier APIs (Delhivery B2C/B2B, Shiprocket, Velocity) and updates order statuses in MongoDB. Runs 5 tracking functions on cron schedules, plus immediate runs on server startup. Production only (ENV=PROD).

## Entry Points
- **`main.go:81-183`** — Cron registration, per-tracker mutexes, immediate startup goroutines
- **`services/order_tracking.go`** — All tracking functions (~3000 lines)
- **`utils/tracking_helper.go`** — MongoDB filters, `UpdateOrderTrackingWithHistory()`
- **`utils/delhivery_status_mapping.go`** — Delhivery status normalization + mapping (B2C StatusType, primary, B2B, fallback)
- **`utils/shiprocket_status_mapping.go`** — Shiprocket status mapping
- **`utils/velocity_status_mapping.go`** — Velocity status mapping
- **`utils/delhivery_response_helper.go`** — B2B API call, response parsing, status extraction
- **`services/send_slack.go`** — Slack notifications (lost/destroy/booking_cancelled/B2B same-day delivery)
- **`services/tracking_logger.go`** — Buffered alert logging → S3 upload
- **`services/tracking_query_logger.go`** — Query details → S3 + Slack notification

## Tracking Functions

| Function | Cron | Max Orders | API Style | Workers | Courier |
|----------|------|-----------|-----------|---------|---------|
| `TrackDelhiveryAndUpdateOrders` (B2C) | `*/8 * * * *` | 36,000 | Batch GET (50 AWBs) | 50 shipment workers | Delhivery |
| `TrackDelhiveryB2BAndUpdateOrders` | `*/8 * * * *` | 480 | Single LR per GET | 3 (env: `B2B_TRACKING_WORKERS`) | Delhivery B2B |
| `TrackShiprocketAndUpdateOrders` | `*/8 * * * *` | All | Batch POST (50 AWBs) | 50 shipment workers | Shiprocket |
| `TrackVelocityAndUpdateOrders` | `*/8 * * * *` | 10,000 | Batch POST (45 AWBs) | Concurrent batches | Velocity |
| `TrackDelhiveryFallbackOrders` | `0 */2 * * *` | All | Single AWB GET (5s delay) | 3 (env: `FALLBACK_TRACKING_WORKERS`) | Delhivery Fallback |

All 5 run immediately at startup via `go functionName()`. Each guarded by per-tracker `sync.Mutex` with `TryLock()` — overlapping cron skips.

---

## Courier API Details

### Delhivery B2C
- **Endpoint:** `DELHIVERY_URL` env var, GET, `?waybill=awb1,awb2,...`
- **Auth:** `DELHIVERY_AUTH_TOKEN_B2C` header
- **Batch:** 50 AWBs per request
- **Rate limit:** 750 req/5min (uses 700 to stay safe)
- **Response key:** `ShipmentData[].Shipment` → Status.Status, Status.StatusType (UD/RT/DL)
- **Status mapping:** `MapDelhiveryB2CStatusWithType()` — uses StatusType for direction awareness

### Delhivery B2B
- **Endpoint:** `https://ltl-clients-api.delhivery.com/lrn/track`, GET, `?lrnum=LR_NUMBER`
- **Auth:** JWT Bearer token from `https://ltl-clients-api.delhivery.com/ums/login`
- **Token management:** Thread-safe `B2BTokenManager`, auto-refresh on 401 (30s min interval)
- **Rate limit:** 500 req/5min (uses 480)
- **Response key:** `data.status`, `data.wbns[].status`

### Shiprocket
- **Endpoint:** `https://apiv2.shiprocket.in/v1/external/courier/track/awbs`, POST, `{"awbs": [...]}`
- **Auth:** Bearer token via `GetShiprocketToken()`, auto-refresh on 401
- **Batch:** 50 AWBs per request
- **Response:** Map keyed by AWB → `tracking_data.shipment_track[0].current_status`
- **Fallback status:** `shipment_track_activities[0].sr-status-label`

### Velocity
- **Auth:** POST `https://shazam.velocity.in/custom/api/v1/auth-token` with username/password → token
- **Tracking:** POST `https://shazam.velocity.in/custom/api/v1/order-tracking` with `{"awbs": [...]}`
- **Batch:** 45 AWBs (50 limit, 45 for safety)
- **Response:** `result.{AWB}.tracking_data` — AWB is the map key
- **Status extraction:** `shipment_status` field, fallback to `shipment_track[0].current_status`

### Delhivery Fallback
- **Endpoint:** `DELHIVERY_FALLBACK_URL` env var, GET, `?wbn=AWB` (note: `wbn` not `waybill`)
- **Auth:** Same as B2C
- **Rate:** Single AWB, 5-second delay between calls per worker
- **End state detection:** Response messages about "end state 7+ days ago" or "invalid AWB" → sets `being_tracked=false`

---

## Tracking Filters (`utils/tracking_helper.go`)

### Common Base Filter
```
order_status: {$in: [4,5,6,7,8,9,11,14,16,51,26,25]}   (or TRACKING_STATUS env)
isBooked.timestamp: {$gte: now - BUFFER_PERIOD days}      (default 31)
```

### B2C (`GetB2CTrackingFilter`)
- Base + `awb_number` exists, non-empty + `delhivery_lr_number` absent/null/empty
- Tracker adds `bookingCourier: /Delhivery/i`
- Source: `{$in: PrimarySourcesList}` (shopify, woocommerce, Outbound, other, Retailer)

### B2B (`GetDelhiveryB2BTrackingFilter`)
- Base + `delhivery_lr_number` exists, non-empty
- Source: PrimarySourcesList
- bookingCourier: /Delhivery/i

### Shiprocket
- `GetB2CTrackingFilter()` + `bookingCourier: /Shiprocket/i`

### Velocity
- `GetB2CTrackingFilter()` + `bookingCourier: /velocity/i` (case-insensitive regex)

### Fallback
- Like B2C but `source: {$nin: PrimarySourcesList}` + `being_tracked: true or missing`

Sort: `last_tracking_time: 1` (ascending — oldest tracked first)

---

## Status Mapping

### Canonical Statuses
pending, in transit, out_for_delivery, delivered, rto, rto in transit, lost, destroy, booking_cancelled, ndr

### Delhivery B2C (StatusType-based)
**File:** `utils/delhivery_status_mapping.go`

| StatusType | Raw Status | Canonical | Notes |
|-----------|-----------|-----------|-------|
| DL | Delivered | delivered | |
| DL | RTO | rto | |
| UD | Manifested | (no change) | noChange=true |
| UD | Not Picked | booking_cancelled | |
| UD | In Transit | in transit | |
| UD | Pending | pending OR in transit | Based on PickUpDate < today |
| UD | Dispatched | out_for_delivery | |
| RT | In Transit | rto in transit | |
| RT | Pending | rto in transit | |
| RT | Dispatched | rto in transit | |

Status normalization: uppercase, spaces/hyphens → underscores, special chars removed.

### Delhivery Primary/B2B/Fallback (Legacy)
80+ status strings mapped. Key ones:
- NDR → ndr
- MANIFESTED, PICK_UP_PENDING, WAITING_PICKUP → no change
- PICKED_UP, IN_TRANSIT → in transit
- OFD, OUT_FOR_DELIVERY → out_for_delivery
- RTO, RETURNED → rto
- RTO_IN_TRANSIT → rto in transit
- DELIVERED → delivered
- NOT_PICKED → booking_cancelled
- LOST → lost

B2B difference: `noChange=false` for ALL statuses (even unmapped ones fall through to update).

### Shiprocket
**File:** `utils/shiprocket_status_mapping.go` — 70+ statuses
- SHIPPED, PENDING, OUT_FOR_PICKUP → no change
- IN_TRANSIT, PICKED_UP → in transit
- OUT_FOR_DELIVERY → out_for_delivery
- DELIVERED → delivered
- RTO, RTO_INITIATED → rto
- RTO_IN_TRANSIT, RTO_OFD → rto in transit
- CANCELLED, NOT_PICKED → booking_cancelled
- LOST, UNTRACEABLE → lost
- DESTROYED, DISPOSED_OFF → destroy

### Velocity
**File:** `utils/velocity_status_mapping.go`
- CANCELLED, NOT_PICKED → booking_cancelled
- IN_TRANSIT, SHIPPED, PICKED_UP → in transit
- MANIFEST_UPLOADED → no change
- OUT_FOR_DELIVERY → out_for_delivery
- DELIVERED → delivered
- RTO, RTO_INITIATED → rto
- RTO_IN_TRANSIT, RETURN_TO_ORIGIN → rto in transit
- NDR, UNDELIVERED → ndr
- LOST, MISSING → lost
- DAMAGED, DESTROYED → destroy

---

## MongoDB Collections Updated

### `orders` collection
- `last_tracking_time` — always updated (current time)
- `order_status` — status_id from `status` collection
- `courier_status` — raw status string (B2C format: `{statustype}-{raw_status}`)
- `tracking_timestamps.<status_key>` — first occurrence only (`$set` if not exists)
- `order_status_map` — array push of status ObjectID (only if different from last)

### `orderTracking` collection
Updated via `UpdateOrderTrackingWithHistory()`:
- `orderId`, `currentStatus`, `stockShipStatus`
- `History` — array of `{status, timeStamp}` (atomic deduplicated push)

### `status` collection (read-only)
- Queried to get `status_id` for a canonical status name

### `marketplaces` collection (read-only, by RTO webhook)
- Queried for marketplace name by ObjectID

---

## Rate Limiting

Each tracker has independent global state with `sync.RWMutex`:
```go
var delhiveryRateLimitMu sync.RWMutex
var delhiveryRateLimitResumeAt time.Time
```

**429 handling:** Parse `Retry-After` header (supports 7+ header names, seconds/unix/RFC1123 formats), default 5 min.
**403 handling:** 8-minute wait.
**On next cron:** Check `isDelhiveryRateLimited()` → skip if still limited.
**B2B 401:** Token refresh via `B2BTokenManager` (30s min interval between refreshes).

---

## Logging System

### Tracking Logger (`services/tracking_logger.go`)
- 4 separate loggers: B2CLogger, B2BLogger, ShiprocketLogger, FallbackLogger
- Buffered in-memory, max 500KB (`DefaultMaxLogSize`)
- On threshold or flush: upload to S3, send presigned URL (6-day expiry) to Slack
- Fallback: file.io if S3 not configured
- Flush cron: `*/30 * * * *`
- Directory: `/tmp/tracking_logs`
- Slack channel: `C0A4P7ES68Y` (hardcoded)

### Query Logger (`services/tracking_query_logger.go`)
- Logs MongoDB filter, sort, fetched order count
- Creates TXT file → S3 upload → Slack notification with presigned URL (3-day expiry)

### Alert Types
- 🚫 Rate limit hits
- 🚨 API failures
- ⚠️ Missing AWBs, unmapped statuses
- 🔐 Token refresh failures

---

## Side Effects on Status Change

| Status | Action |
|--------|--------|
| booking_cancelled | `SendBookingCancelledSlackNotification()` |
| lost, destroy | `SendLostDestroySlackNotification()` |
| rto, rto in transit | `go TriggerRTOCancelWebhook()` (async, see webhook-system.md) |
| delivered (B2B same-day) | `SendDeliveredB2BSlackNotification()` |

---

## Concurrency Safety

### Per-Tracker Mutex (Fix: Feb 18, 2026)
Each tracker guarded by `sync.Mutex` + `TryLock()`. Startup goroutine + cron can't overlap.

### Atomic History Deduplication (Fix: Feb 18, 2026)
`UpdateOrderTrackingWithHistory`: two-step atomic — upsert with `$setOnInsert` for first entry, then conditional `$push` using `$expr` + `$let` + `$arrayElemAt` to check last status differs.

### B2B Same-Day Delivery Slack (Fix: Feb 18, 2026)
`isBooked.timestamp` decoded as `primitive.DateTime` first, fallback to `time.Time`.

---

## Reference Matching (B2C)
Delhivery returns `ReferenceNo` — compared with `order_id`. Accounts for rebooking suffixes: `-C`, `-C1`, `-C2`, `-k`, `-r`, `-K`, `-R`. Case-insensitive.

---

## Environment Variables

**API Endpoints:** `DELHIVERY_URL`, `DELHIVERY_FALLBACK_URL`, `DELHIVERY_B2B_URL`
**Auth:** `DELHIVERY_AUTH_TOKEN_B2C`, `DELHIVERY_ORIGIN`, `VELOCITY_USERNAME`, `VELOCITY_PASSWORD`
**Config:** `BUFFER_PERIOD` (31), `TRACKING_STATUS` (comma-sep IDs), `B2B_TRACKING_WORKERS` (3), `FALLBACK_TRACKING_WORKERS` (3), `ENV` (PROD to enable)
**Email:** `EMAIL_SENDER`, `EMAIL_PASSWORD`, `EMAIL_RECIPIENTS`
**AWS/S3:** `AWS_S3_BUCKET`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
**Slack:** `SLACK_TOKEN`, `TRACKING_QUERY_SLACK_CHANNEL`, `LOST_DESTROY_SLACK_CHANNEL_ID`, `BOOKING_CANCELED_CHANNEL_ID`

---

## Email Reports
Each tracker sends an email summary after completion via SMTP (Gmail). Contains tracking results table. Recipients from `EMAIL_RECIPIENTS` env var (comma-separated).
