# Inventory-Management-Backend

## Overview
Backend service for inventory and order management at Katyayani Organics.

## Tech Stack
- **Language:** Node.js, Express.js, JavaScript
- **Database:** MongoDB (Mongoose ODM)

## Status
- Active project

## Key Files
- `models/order-model.js` — Order schema (customer embedded doc, marketplace ref, tracking_timestamps, order_status)
- `models/marketPlace-model.js` — Marketplace schema (name, sku_map, warehouses)
- `models/status-model.js` — Status collection (status_id ↔ status name mapping)
- `routes/orders/controllers/orderStatus-controller.js` — cancelOrder, order stage management
- `routes/orders/utils/customerHelper.js` — `update_cancelled_orders_customer()` calls CRM cancel-order API (POST)

## CRM Integration
- `customerHelper.js` sends POST to `CRM_BASED_URL/customer/cancel-order` with `{phone_number, order_id, order_source, order_value, customer_id, status: "cancelled"}`
- Changed from PATCH → POST (Feb 2026) to align with webhook system

## Modules

| Module | Doc | Summary |
|--------|-----|---------|
| [Freebies](modules/freebies.md) | Deep | Rule-based promotional freebie system (json-rules-engine, dual quotas, versioned rules) |
| [Katyayani Associate](modules/katyayani-associate.md) | Deep | Visibility & pricing rule engine for B2B associates (json-rules-engine, field priority scoring, deny-by-default) |
