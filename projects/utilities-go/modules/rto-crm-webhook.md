# RTO → CRM Cancel-Order Webhook (Feb 2026)

## Purpose
When tracking crons detect an order's first RTO status, trigger a webhook to call CRM's cancel-order API so customer metrics (rto_count, total_order_value, etc.) are updated.

## Flow
```
Tracking Cron detects RTO → TriggerRTOCancelWebhook()
  → "only once" check (tracking_timestamps.rto / rto_in_transit)
  → lookupMarketplaceName (MongoDB marketplaces collection)
  → ExecuteWebhookTrigger("order", "", "order.rto", updatedData, buildingData)
  → Webhook system: conditions → payload mapping → auth → POST to CRM
  → CRM /customer/cancel-order processes RTO
```

## Files
| File | What |
|------|------|
| `services/rto_webhook.go` | TriggerRTOCancelWebhook, lookupMarketplaceName |
| `services/webhook_trigger.go` | ExecuteWebhookTrigger (moved from handlers) |
| `services/order_tracking.go` | 5 trigger points (B2C, B2C Primary, B2B, Shiprocket, Fallback) |

## "Only Once" Logic
Before firing, checks `order["tracking_timestamps"]`:
- If `.rto` is already set → skip (order already triggered)
- If `.rto_in_transit` is already set → skip
- If neither set → first RTO → fire webhook

## Trigger Points in order_tracking.go
Added `case "rto", "rto in transit":` at 5 switch/if locations:
1. B2C Delhivery (~line 1443)
2. B2C Delhivery Primary (~line 1658)
3. B2B Delhivery (~line 2089, as else-if)
4. Shiprocket (~line 1162)
5. Fallback Delhivery (~line 2578)

## Webhook Config Required (via API after deployment)
1. **Event:** `order.rto`
2. **Auth:** CRM token-based (auth_url, auth_payload, response_token_path)
3. **Payload mapping:** `phone_number→customer.phone`, `order_id→order_id`, `order_source→source`, `order_value→total_amt`, `status→status`
4. **Conditions:** `tracking_status` operator `in` value `rto,rto in transit` (group 1)
5. **Webhook:** URL = CRM cancel-order endpoint, module = "order", events = [order.rto]

## Related Changes (Across Repos)
- **CRM-Backend:** cancel-order route changed from PATCH → POST (both .ts and .cjs)
- **Inventory-Management-Backend:** customerHelper.js axios.patch → axios.post
- **utilities-go JSON schema:** Added `status` field, removed restrictive enums from `tracking_status`, `source`, `courier_name`
- **validate_payload_fields.go:** Added `"status": FieldString` to orderFieldPaths
