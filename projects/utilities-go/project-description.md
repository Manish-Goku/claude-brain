# Utilities-Go

## Overview
Golang-based utility services for Katyayani Organics.

## Tech Stack
- **Language:** Golang, Gin framework
- **Databases:** MongoDB (orders, marketplaces, status), PostgreSQL (webhooks via GORM)
- **Key Packages:** `go.mongodb.org/mongo-driver`, `gorm.io/gorm`, `gorm.io/datatypes`, `github.com/xeipuuv/gojsonschema`

## Project Structure
```
utilities-go/
├── main.go
├── database/           # MongoDB + PostgreSQL connections
│   ├── mongo.go        # database.Collection("name") helper
│   └── postgress.go    # GormDB, AutoMigrate for webhook models
├── models/
│   ├── cron_model.go
│   └── webhook_models/ # Webhook, Auth, Payload, Condition, Log, Event, Source
├── handlers/
│   ├── cron_handler.go
│   └── webhook_handlers/  # Full CRUD for all resources + trigger + retry + logs
├── services/
│   ├── webhook.go          # ExecuteWebhook (HTTP caller), auth helpers
│   ├── webhook_trigger.go  # ExecuteWebhookTrigger (core trigger logic, conditions, payload building)
│   ├── rto_webhook.go      # TriggerRTOCancelWebhook, lookupMarketplaceName
│   ├── order_tracking.go   # Tracking crons (Delhivery B2C/B2B, Shiprocket, Fallback)
│   ├── send_slack.go       # Slack notifications for lost/destroy/booking_cancelled
│   └── tracking_alerts.go  # Alert logging helpers
├── utils/
│   ├── schema_manager.go             # JSON schema validation singleton (gojsonschema)
│   ├── schemas/order.json            # Order module JSON schema
│   ├── tracking_helper.go            # MongoDB tracking filters
│   ├── tracking_logger.go            # Buffered S3 log upload
│   ├── tracking_query_logger.go      # Query audit logging
│   ├── delhivery_status_mapping.go   # Courier → canonical status mapping
│   ├── shiprocket_status_mapping.go  # Shiprocket status mapping
│   └── cron_helper.go                # Cron scheduler
├── routes/
│   └── webhookRoutes.go    # All webhook + meta routes (full CRUD)
└── postman_collections/
    └── Webhook_API.postman_collection.json  # v2.0 — all routes
```

## Status
- Active project

## Key Architecture
- **Webhook System:** Configurable webhooks with events, sources, conditions (AND/OR groups), payload mapping (dot notation), dual auth (headers or token-based), retry, logging, JSON schema validation
- **Conditions mandatory:** Cannot create webhook without >=1 condition (validated in CreateWebhook)
- **ExecuteWebhookTrigger()** lives in `services/` (moved from handlers Feb 2026 to avoid circular deps) — callable from any service without HTTP context
- **Dual validation:** JSON schema (active, validates data types/formats) + Field path registry (inactive TODO)
- **Tracking Crons:** 4 tracking functions poll courier APIs and update order statuses in MongoDB
- **RTO → CRM Integration (Feb 2026):** When tracking detects first RTO status, fires `order.rto` webhook to CRM cancel-order API via `TriggerRTOCancelWebhook()`
- **Delete safety:** All delete handlers check FK/M2M references before allowing deletion

## Modules
- [Webhook System](./modules/webhook-system.md) — Complete webhook feature reference (models, routes, flows, validation, file map)
- [RTO-CRM Webhook](./modules/rto-crm-webhook.md) — RTO webhook trigger integration
- [Order Tracking](./modules/order-tracking.md) — Tracking crons, B2B/B2C/Shiprocket/Fallback, status mapping, known bugs
