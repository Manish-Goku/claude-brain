# Webhook System — Complete Reference

## Architecture Overview
Configurable webhook system with event-driven triggers, conditional execution, dynamic payload mapping, dual auth mechanisms, retry support, and JSON schema validation.

**Databases:** PostgreSQL (GORM) for webhook config, MongoDB for order data
**Packages:** `gorm.io/gorm`, `gorm.io/datatypes`, `github.com/xeipuuv/gojsonschema`

---

## Database Models (PostgreSQL, GORM AutoMigrate)

### Webhook (`webhook_models/webhook.go`)
```
ID, URL, AuthID*, PayloadID*, RegisteredBy, Description,
IsActive (default:true), ModuleName, RetryEnabled (default:true),
MaxRetries (>=0), CreatedAt, UpdatedAt
Relations:
  Auth        → *WebhookAuth (FK: AuthID)
  Payload     → *WebhookPayload (FK: PayloadID)
  Events      → []WebhookEvent (M2M: webhook_subscribed_events)
  Sources     → []WebhookSource (M2M: webhook_allowed_sources)
  Conditions  → []WebhookCondition (HasMany)
```

### WebhookAuth (`webhook_models/webhook_auth.go`)
```
ID, Headers (jsonb), AuthURL, AuthPayload (jsonb), ResponseTokenPath, CreatedAt
```
Two modes (mutually exclusive):
- **Header-based:** `Headers` JSON with key-value pairs injected directly
- **Token-based:** `AuthURL` + `AuthPayload` + `ResponseTokenPath` — fetches token via POST, extracts via dot path, injects as Bearer

### WebhookPayload (`webhook_models/webhook_payload.go`)
```
ID, ModuleName, PayloadMapping (jsonb), IncludeEventMeta (default:false), CreatedAt
```
PayloadMapping format: `{ "outgoing_key": "dot.notation.path" }`
Example: `{ "phone_number": "customer.phone", "order_value": "total_amt" }`

### WebhookCondition (`webhook_models/webhook_condition.go`)
```
ID, WebhookID, ModuleName, FieldPath, Operator, Value, ConditionGroup (default:1), CreatedAt
```
- **FieldPath:** dot notation (e.g., `customer.phone`, `tracking_status`)
- **Operators:** `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `in`
- **Condition Logic:** AND within same group, OR across groups

### WebhookLog (`webhook_models/webhook_log.go`)
```
ID, WebhookID, OrderID, Success (default:false), AttemptCount (default:0),
MaxAttempts, TotalTimeTakenMs, SuccessPayloadSnapshot (jsonb),
AuthID*, FailedLogs (jsonb), CreatedAt, CompletedAt*
```
FailedLog (embedded in FailedLogs JSON array):
```
AttemptNumber, FailureReason, HTTPStatus, TimeTakenMs, AttemptedAt, PayloadSnapshot
```

### WebhookEvent (`webhook_models/webhook.go`)
```
ID, Name (unique)
```

### WebhookSource (`webhook_models/webhook.go`)
```
ID, Name (unique)
```

### Join Tables (GORM auto-created)
- `webhook_subscribed_events` — Webhook ↔ WebhookEvent
- `webhook_allowed_sources` — Webhook ↔ WebhookSource

---

## API Routes (`routes/webhookRoutes.go`)

### `/webhooks` (JWKS auth)
| Method | Path | Handler | Notes |
|--------|------|---------|-------|
| POST | `/` | CreateWebhook | Conditions mandatory (>=1) |
| GET | `/` | ListWebhooks | ?module_name, ?is_active filters |
| GET | `/:id` | GetWebhook | Preloads Events, Sources, Conditions |
| PUT | `/:id` | UpdateWebhook | |
| DELETE | `/:id` | DeleteWebhook | |
| POST | `/:id/conditions` | CreateCondition | |
| GET | `/:id/conditions` | ListConditions | Ordered by condition_group |
| PUT | `/:id/conditions/:conditionId` | UpdateCondition | |
| DELETE | `/:id/conditions/:conditionId` | DeleteCondition | |
| GET | `/:id/logs` | ListWebhookLogs | Ordered by created_at DESC |
| POST | `/logs/:id/retry` | RetryWebhook | Custom payload or last failed snapshot |
| POST | `/trigger` | TriggerWebhooks | Manual HTTP trigger |

### `/webhook-meta` (JWKS auth)
| Method | Path | Handler | Notes |
|--------|------|---------|-------|
| POST | `/auth` | CreateAuth | |
| GET | `/auth` | GetAuths | Paginated (page, limit) |
| GET | `/auth/:id` | GetAuth | |
| PUT | `/auth/:id` | UpdateAuth | |
| DELETE | `/auth/:id` | DeleteAuth | Checks webhook FK refs |
| POST | `/payload` | CreatePayload | Validates mapping against schema |
| GET | `/payload` | GetPayloads | Paginated, ?module_name filter |
| GET | `/payload/:id` | GetPayload | |
| PUT | `/payload/:id` | UpdatePayload | Re-validates mapping |
| DELETE | `/payload/:id` | DeletePayload | Checks webhook FK refs |
| POST | `/events` | CreateEvent | |
| GET | `/events` | ListEvents | |
| DELETE | `/events/:id` | DeleteEvent | Checks M2M refs |
| POST | `/sources` | CreateSource | |
| GET | `/sources` | ListSources | |
| DELETE | `/sources/:id` | DeleteSource | Checks M2M refs |

---

## Core Execution Flow

### 1. Trigger Entry Point
```
ExecuteWebhookTrigger(moduleName, source, event, updatedData, buildingData)
  → services/webhook_trigger.go
```
- `updatedData` — used for condition evaluation
- `buildingData` — used for payload building (can be superset of updatedData)

Can be called:
- Via HTTP: `POST /webhooks/trigger` → handler calls services.ExecuteWebhookTrigger
- Programmatically: Any service imports and calls directly (e.g., `rto_webhook.go`)

### 2. Webhook Filtering
```
filterWebhooksByEventAndSource(webhooks, event, source)
```
- Fetches all active webhooks for module (with preloads: Events, Sources, Conditions, Payload, Auth)
- Filters by event name match in Events slice
- Filters by source name match in Sources slice (empty sources = allow all)

### 3. Condition Evaluation
```
EvaluateConditions(conditions, updatedData)
```
- Groups conditions by `condition_group`
- **OR across groups:** if ANY group is fully satisfied → true
- **AND within group:** ALL conditions in group must pass
- Empty conditions → true (but CreateWebhook now requires >=1)

Operators:
- `==`, `!=` — string comparison
- `>`, `<`, `>=`, `<=` — numeric (converts to float64)
- `contains` — substring match
- `in` — comma-separated list check

### 4. Payload Building
```
BuildPayload(payloadConfig, buildingData, event, webhookID)
```
- Unmarshals PayloadMapping JSON
- For each mapping `"outKey": "dot.path"`, extracts value from buildingData via ExtractValueFromPath
- Optionally adds `_event_meta` (event name, webhook_id, timestamp)

### 5. Validation
```
schemaManager.ValidatePayload(moduleName, payload)
```
- Validates built payload against JSON schema (`utils/schemas/<module>.json`)
- Uses `gojsonschema` library
- Graceful fallback: if schema doesn't exist for module, allows payload
- Called during: `executeSingleWebhook` (trigger flow) and `validateCustomPayload` (retry with custom payload)

### 6. Execution
```
ExecuteWebhook(url, payload, auth)  → services/webhook.go
```
- HTTP POST with JSON body
- Applies auth (headers or Bearer token)
- Validates 2xx response

### 7. Logging
- Creates `WebhookLog` entry per webhook execution
- On success: sets Success=true, stores SuccessPayloadSnapshot, sets CompletedAt
- On failure: appends to FailedLogs JSON array with attempt details

---

## Validation System (Dual Layer)

### Layer 1: JSON Schema (`utils/schemas/order.json`)
- **Status:** ACTIVE
- **Used by:** `SchemaManager.ValidatePayload()` in `utils/schema_manager.go`
- **When:** During webhook execution (trigger + retry)
- **What it validates:** Data types, formats (email, phone, date-time, pincode), patterns, ranges (min >= 0), enums (payment_mode, currency_code, order_status)
- **Schema:** Embedded via `//go:embed schemas/*.json`, cached in singleton SchemaManager
- **Graceful fallback:** Unknown modules pass validation

Schema fields with constraints:
- `payment_mode` enum: `["COD", "PREPAID", "POSTPAID"]`
- `currency_code` enum: `["INR", "USD", "EUR"]`
- `order_status` integer enum: `[0, 1, 2, 3, 4, 5]`
- `customer.phone` pattern: `^[0-9]{10}$`
- `billing_addr.pincode` / `shipping_addr.pincode` pattern: `^[0-9]{6}$`
- `customer.email` format: email
- `tracking_status` — free string (enum removed Feb 2026, values are dynamic from tracking crons)
- `source` — free string (enum removed Feb 2026, values are marketplace names)
- `courier_name` — free string (enum removed Feb 2026)
- `status` — free string (added Feb 2026 for RTO webhook)

### Layer 2: Field Path Registry (`models/webhook_models/validate_payload_fields.go`)
- **Status:** INACTIVE (ValidateFieldPath is TODO — always returns nil)
- **Purpose (intended):** Validate that payload mapping field paths exist in module schema
- **What it defines:** `ModuleFieldRegistry` → `orderFieldPaths` map of allowed dot paths with types
- **Currently used by:** `validatePayloadMapping()` in payload handler — but calls `ValidateFieldPath()` which is a no-op
- **Future use:** Wire up proper field path traversal to reject invalid mapping paths

---

## Retry System

### Endpoint: `POST /webhooks/logs/:id/retry`
1. Fetch WebhookLog by ID
2. Fetch parent Webhook with Payload preload
3. Check FailedLogs exist
4. **Custom payload path:** If request body has `payload`, validate against JSON schema + mapping structure
5. **Snapshot path:** Otherwise use last failed attempt's PayloadSnapshot
6. Execute webhook
7. Append new FailedLog entry (success or failure)
8. On success: set log.Success=true, store SuccessPayloadSnapshot, set CompletedAt
9. Save updated log

---

## Handler Details

### CreateWebhook Validation (Feb 2026)
- `conditions` array is **mandatory** (>=1 required)
- Each condition validated: `field_path`, `operator`, `value` non-empty
- `operator` must be in: `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `in`
- `module_name` auto-filled on each condition from webhook's module_name
- GORM `Create(&input)` auto-creates has-many conditions atomically

### Delete Handlers — Foreign Key Safety
- **DeleteAuth:** Checks `webhooks` table for `auth_id` reference
- **DeletePayload:** Checks `webhooks` table for `payload_id` reference
- **DeleteEvent:** Checks `webhook_subscribed_events` M2M table
- **DeleteSource:** Checks `webhook_allowed_sources` M2M table

### Pagination (Auth List, Payload List)
- Query params: `page` (default 1), `limit` (default 20)
- Returns: `{ data: [...], pagination: { page, limit, total, total_pages } }`

---

## File Map

| File | Purpose | Line Count |
|------|---------|------------|
| `models/webhook_models/webhook.go` | Webhook, Event, Source structs | 40 |
| `models/webhook_models/webhook_auth.go` | WebhookAuth struct | 18 |
| `models/webhook_models/webhook_payload.go` | WebhookPayload struct | 17 |
| `models/webhook_models/webhook_condition.go` | WebhookCondition struct | 16 |
| `models/webhook_models/webhook_log.go` | WebhookLog, FailedLog structs | 34 |
| `models/webhook_models/validate_payload_fields.go` | Field registry (inactive) | 87 |
| `services/webhook.go` | ExecuteWebhook, auth helpers | 152 |
| `services/webhook_trigger.go` | ExecuteWebhookTrigger, conditions, payload | 442 |
| `services/rto_webhook.go` | TriggerRTOCancelWebhook | 97 |
| `handlers/webhook_handlers/webhook_handler.go` | CRUD + Retry | 320 |
| `handlers/webhook_handlers/webhook_auth_handler.go` | Auth CRUD | 184 |
| `handlers/webhook_handlers/webhook_payload_handler.go` | Payload CRUD | 197 |
| `handlers/webhook_handlers/webhook_conditon_handler.go` | Condition CRUD | 77 |
| `handlers/webhook_handlers/webhook_trigger_handler.go` | HTTP trigger adapter | 38 |
| `handlers/webhook_handlers/webhook_logs_handler.go` | Log listing | 27 |
| `handlers/webhook_handlers/event_and_source_handler.go` | Event + Source CRUD | 88 |
| `routes/webhookRoutes.go` | All route registration | 54 |
| `utils/schema_manager.go` | JSON schema validation singleton | 116 |
| `utils/schemas/order.json` | Order module JSON schema | ~250 |
| `database/postgress.go` | PostgreSQL + GORM init, AutoMigrate | 96 |
| `postman_collections/Webhook_API.postman_collection.json` | Full Postman collection v2.0 | 684 |

---

## Postman Collection (`postman_collections/Webhook_API.postman_collection.json`)
Version 2.0.0 — updated Feb 2026 with all CRUD routes.

Variables: `BASE_URL` (http://localhost:8080), `JWT_TOKEN`

Folders: Webhooks (6), Conditions (4), Logs (2), Auth (6), Payload (5), Events (3), Sources (3)

---

## Integration Points — How Webhooks Are Triggered

### RTO Webhook (services/rto_webhook.go)
- `TriggerRTOCancelWebhook(order bson.M, mappedStatus, orderIDStr string)` — called as goroutine from tracking crons
- One-time guard: checks `tracking_timestamps.rto` / `rto_in_transit` already set → skip
- Looks up marketplace name from MongoDB `marketplaces` collection via ObjectID
- Flattens `order["customer"]` bson.M → `customerData` map for dot-notation
- Calls `ExecuteWebhookTrigger("order", "", "order.rto", updatedData, buildingData)`
- Called from 5 places in `order_tracking.go` (B2C, B2C Primary, B2B, Shiprocket, Fallback) inside `case "rto", "rto in transit":`

### Tracking Cron Integration (services/order_tracking.go)
- Tracking crons run every 8 min (fallback every 2 hrs), production only
- Each cron has a `sync.Mutex` preventing overlapping runs
- When a courier status maps to "rto" or "rto in transit", fires `go TriggerRTOCancelWebhook(order, mappedStatus, orderIDStr)`
- The `go` keyword means it's fire-and-forget (non-blocking to the tracking loop)

### Manual Trigger (POST /webhooks/trigger)
- HTTP endpoint for testing/manual use
- Request body: `{ module_name, source, event, updated_data, building_data }`

---

## HTTP Execution Details (services/webhook.go)
- HTTP client timeout: **15 seconds**
- Method: **POST** (hardcoded)
- Content-Type: **application/json**
- Auth applied via `applyWebhookAuth(client, req, auth)`:
  - Static headers: unmarshals JSON → sets each header
  - Token-based: `fetchAuthToken()` → POST to AuthURL with AuthPayload → extracts token via dot-path from response → sets `Authorization: Bearer <token>`
- Success: HTTP 2xx; anything else returns error

---

## Known TODOs
1. `ValidateFieldPath()` in schema_manager.go — implement proper JSON schema traversal for field path validation
2. `GetSchemaDescription()` — returns placeholder, implement proper field description retrieval
3. `regex` operator — referenced in code comments but not implemented in compareValues
4. Webhook HTTP method — hardcoded POST, no support for PATCH/PUT/DELETE
5. Automatic retry — `RetryEnabled`/`MaxRetries` fields exist but auto-retry is NOT implemented in trigger flow; only manual retry via API
