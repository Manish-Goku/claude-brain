# Freebie Module — Inventory-Management-Backend

## Overview
Rule-based promotional freebie system using `json-rules-engine`. Evaluates customer eligibility based on configurable conditions (order value, location, payment mode, cart contents, lifetime metrics) and manages distribution via agent quotas and customer recurrence limits.

## Key Files

| File | Purpose |
|------|---------|
| `routes/freebies/route.js` | Route registration (auto-loaded) |
| `routes/freebies/controllers/controller.js` | 11 API endpoints |
| `routes/freebies/utils/freebieRuleEngine.js` | Core evaluation logic (json-rules-engine) |
| `routes/freebies/validators/*.js` | 8 express-validator files |
| `models/freebieRuleModel.js` | Rule schema (conditions, priority, versioning) |
| `models/freebieUsageLogModel.js` | Usage tracking per customer/order |
| `models/agentFreebieQuotaModel.js` | Per-agent distribution limits |
| `models/promotionalItems.js` | Freebie items (product or combo type) |
| `routes/freebies/documentation/` | 9 detailed markdown docs |

## Models (4)

### FreebieRule (`freebie_rules`)
- `rule_name` (immutable), `freebie_skus[]`, `freebie_refs[]` (→ promotional_item)
- `conditions` (json-rules-engine format: nested `all`/`any` with fact/operator/value)
- `priority` (higher = shown first), `is_active`, `version` (auto-incremented on update)
- `event.type` (default: "FREEBIE_ELIGIBLE"), `event.params`
- `considered_as_log` — marks old versions (versioning history)
- Indexes: `freebie_skus`, `is_active + priority`, `freebie_refs`

### FreebieUsageLog (`freebie_usage_logs`)
- `customer_number`, `customer_ref` (→ Customer), `freebie_sku`, `freebie_ref`
- `order_id`, `quantity`, `agent_id`, `issued_at`, `status`
- Status flow: `pending → confirmed → delivered` or `confirmed → cancelled`

### AgentFreebieQuota (`agent_freebie_quotas`)
- `agent_id`, `freebie_sku`, `freebie_ref`, `quota_limit`, `quota_used`, `quota_remaining`
- `period` (monthly/quarterly/yearly/lifetime), `period_start`, `period_end`
- `usage_history[]` — audit trail per distribution
- Auto-resets quota_used when period expires

### PromotionalItem (`promotional_items`)
- `sku` (unique, uppercase), `name`, `type` (product/combo), `category`
- `dimensions`, `weight`, `images[]`, `available_quantity`
- `items[]` — combo components: `{ product: ObjectId, item_quantity: Number }`
- `agent_quota_config` — `{ enabled, type, limit, period }`
- `customer_quota_config` — `{ enabled, type, limit, period }`

## Evaluation Flow

```
check-eligibility request
  → build_facts() — merge request payload + DB aggregation (lifetime order value/count from delivered orders only, status=15)
  → Load active rules sorted by priority DESC
  → For each rule:
      → json-rules-engine evaluates conditions against facts
      → If pass: for each freebie in rule:
          → check_inventory_availability() — product: direct qty, combo: min(component_qty / item_qty)
          → check_freebie_recurrence() — customer_quota_config (lifetime/days/weekly/monthly/quarterly/yearly)
          → check_agent_quota() — agent quota doc lookup, auto-reset if period expired
      → Classify as eligible or ineligible (with reason codes)
  → Return { eligible_freebies[], ineligible_freebies[] }
```

## Available Facts for Rules

| Fact | Source |
|------|--------|
| `current_order_value_before_discount` | Request payload |
| `current_order_value_after_discount` | Request payload |
| `delivery_state`, `delivery_city`, `delivery_pincode` | Request payload |
| `delivery_zone`, `delivery_cluster` | Request payload |
| `payment_mode` | Request payload (lowercase) |
| `agent_id` | Request payload |
| `cart_items` | Request payload (array with sku + quantity) |
| `customer_lifetime_order_value` | DB aggregation (delivered orders) |
| `customer_lifetime_order_count` | DB aggregation (delivered orders) |

## Custom Operators
- `notIn` — factValue not in jsonValue array
- `containsSku` — cart_items contains `{ sku, min_quantity }` match

## API Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/api/freebies/check-eligibility` | No | Evaluate customer freebie eligibility |
| POST | `/api/freebies/rules` | Yes | Create freebie rule |
| GET | `/api/freebies/rules` | Yes | List rules (paginated, filter: is_active) |
| GET | `/api/freebies/rules/:id` | Yes | Get single rule (populated freebies) |
| PUT | `/api/freebies/rules/:id` | Yes | Update rule (versioned if conditions change) |
| POST | `/api/freebies/agent-quota` | Yes | Create agent quota |
| GET | `/api/freebies/agent-quota` | Yes | List quotas (filter: agent_id, freebie_sku) |
| PUT | `/api/freebies/agent-quota/:id` | Yes | Update quota |
| DELETE | `/api/freebies/agent-quota/:id` | Yes | Delete quota |
| POST | `/api/freebies/record-usage` | Yes | Record freebie distribution |
| GET | `/api/freebies/customer-usage` | Yes | Customer usage logs (paginated) |
| GET | `/api/freebies/agent-usage` | Yes | Agent usage logs (paginated) |

## Rule Versioning
On update: if conditions/logic change (not just is_active toggle):
1. Old rule → `is_active: false`, `considered_as_log: true`
2. New rule created with `version: old_version + 1`
3. Requires `comment` field explaining the change

## Integration Pattern
Standalone module — NOT automatically integrated into order creation.
Expected flow: check-eligibility → customer/agent selects → order created → record-usage → status updates with order lifecycle.

## Ineligibility Reason Codes
- `OUT_OF_STOCK` — zero inventory
- `ALREADY_RECEIVED` — customer recurrence limit hit
- `AGENT_QUOTA_EXCEEDED` — agent distribution limit hit
- `ALL_FREEBIES_FAILED` — no freebie available under the rule
