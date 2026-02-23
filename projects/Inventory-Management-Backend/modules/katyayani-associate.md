# Katyayani Associate Module — Inventory-Management-Backend

## Overview
Visibility & pricing rule engine for B2B associates. Controls which bulk products an associate can see and at what price/MOQ using `json-rules-engine`. **Deny by default** — no matching rules = zero products.

## Key Files

| File | Purpose |
|------|---------|
| `routes/katyayani-associate/route.js` | 19 endpoints (all auth-required) |
| `routes/katyayani-associate/controllers/controller.js` | All endpoint handlers |
| `routes/katyayani-associate/models/partner.js` | Associate profile schema (`associates` collection) |
| `routes/katyayani-associate/models/associateRules.js` | Rule definitions (`associate_partner_rules`) |
| `routes/katyayani-associate/models/fieldsPriority.js` | Fact → priority weight (`partner_field_priorities`) |
| `routes/katyayani-associate/models/clusterDefinition.js` | Cluster → pincodes (`associate_cluster_definitions`) |
| `routes/katyayani-associate/utils/visibilityRuleEngine.js` | Core visibility logic |
| `routes/katyayani-associate/utils/pricingRuleEngine.js` | Core pricing logic |
| `routes/katyayani-associate/utils/helper.js` | Shared utilities (engine creation, fact extraction) |
| `routes/katyayani-associate/validators/` | 9 express-validator files |

## Models (4)

### Associate (`associates`)
Profile fields that become rule facts: `user_id`, `segment_type` (p2p_regulated/p2p_unregulated/pest_control/tender/b2b_bulk), `tier` (bronze/silver/gold/diamond), `risk_level`, `has_sales_license`, `total_purchases`, `coin_balance`, `compliance_score`, `is_approved`, `catalog_mode`. Also has `documents[]` subdoc, `onboarding_status`, `pii_id`, `contact_id`.

### AssociateRules (`associate_partner_rules`)
4 rule types: `category`, `bulk`, `cluster`, `moq`. Each has `conditions` (json-rules-engine format), `priority`, `is_active`, `considered_as_log`, `version`, `rule_name` (immutable). Category/cluster rules use `categories[]` (with `~` prefix for exclusion). Bulk rules use `sku` field. Cluster rules add `cluster_name` + `bulks[]`. MOQ rules add `pricing: {price, moq}`.

### FieldPriority (`partner_field_priorities`)
Maps fact names to priority weights. `user_id` priority auto-enforced to be > sum of all others. Used to score competing rules — highest score wins.

### ClusterDefinition (`associate_cluster_definitions`)
Maps `cluster_name` → `pincodes[]`. Cluster rules only activate when user's pincode belongs to the referenced cluster.

## Evaluation Flow

```
Request → build_user_facts(user_id, pincode) → flat facts object
  → evaluate_visibility(facts) → loads active category/bulk/cluster rules
    → For each rule: json-rules-engine evaluates conditions
    → Passing rules → include/exclude sets (categories + bulks)
    → ~prefix = exclusion (always wins over inclusion)
  → resolve_visible_bulks() → MongoDB query from include/exclude sets → paginated results
  → evaluate_pricing(facts, bulks) → loads active MOQ rules
    → Score by field priority, highest-score match wins per bulk
    → Returns Map<bulk_sku, {price, moq}>
```

## Priority Resolution
`score = sum of field_priority[fact] for each fact in rule.conditions`
- `user_id` always highest → user-specific rules override everything
- When multiple rules match same bulk, highest score wins

## Rule Versioning
Logic updates: deactivate old versions (`considered_as_log: true`), create new version+1. Requires `comment`.
Toggle-only (`is_active`): in-place update, no versioning.

## API Endpoints

### User-Facing
| Method | Path | Description |
|--------|------|-------------|
| GET | `/bulks` | Paginated catalog with visibility + pricing. Query: `pincode, page, limit, search` |
| GET | `/bulks/:sku` | Single bulk detail with pricing. Query: `pincode` |
| GET | `/category/tree` | Category tree filtered by visibility. Query: `pincode` |
| GET | `/product/details` | Product details by SKU (queries PostgreSQL for packaging). Query: `sku` |

### Admin: Rules CRUD
POST/GET/GET/:id/PUT/:id/DELETE/:id on `/rules` — versioned updates, filter by `is_active`, `type`

### Admin: Field Priorities
POST/GET/PUT/:field_name/DELETE/:field_name on `/field-priorities` — cannot delete `user_id`

### Admin: Clusters
POST/GET/GET/:name/PUT/:name/DELETE/:name on `/clusters` — soft delete

## Utils Detail

### helper.js
- `is_only_is_active_updated(payload)` — detects toggle-only updates
- `create_engine_with_operators()` — json-rules-engine with custom ops: `notIn`, `containsAny`
- `extract_facts_from_conditions(conditions)` — walks condition tree, returns Set of fact names

### visibilityRuleEngine.js
- `build_user_facts(user_id, pincode)` — loads associate profile → flat facts
- `evaluate_visibility(user_facts)` — evaluates all active visibility rules, returns include/exclude sets
- `resolve_visible_bulks(visibility_result, options)` — converts to MongoDB query, paginated
- `resolve_single_bulk(sku, visibility_result)` — checks if specific bulk is visible
- `get_visible_category_ids(visibility_result)` — ObjectIds of visible categories
- 5-min field priority cache with `invalidate_field_priorities_cache()`

### pricingRuleEngine.js
- `evaluate_pricing(user_facts, bulks)` — loads MOQ rules, scores by field priority, highest match wins per bulk. Targets: specific SKU, category-wide, or global.

## Relationship to Freebie Module
Mirrors freebie rule engine pattern but adapted for visibility/pricing. Same versioning, toggle-update, and engine patterns. Key difference: condition validation is dynamic (from DB) vs freebie's hardcoded `ALLOWED_FACTS`.
