# Pincode Case Pricing System

## Overview
Zone-based pricing system that overrides default case pricing for specific geographical zones (identified by pincodes). Managed via utilities-go REST API, consumed by Inventory-Management-Backend's search product API via shared MongoDB.

**Collections:** `pincode_case_pricing`, `pincode_case_audit_log` (MongoDB)
**Depends on:** `cases` collection (shared with Inventory-Management-Backend)

---

## MongoDB Models

### pincode_case_pricing
```
ID, CaseSKU (→ cases.case_id), Zone, UOM (l/ml/gm/kg), PincodesArray []string,
Pricing []PricingDetail, IsActive (default:false), CreatedAt, UpdatedAt, CreatedBy, UpdatedBy
```

**PricingDetail:**
```
Quantity (int), UnitPrice (float64), UOMUnitPrice (float64), UnitCasePrice (float64), Default (bool)
```

**Indexes:**
- Unique compound: `(case_sku, pincodes_array)` — prevents same pincode in multiple zones per SKU
- Compound: `(case_sku, zone)` — fast zone lookups
- Compound: `(is_active, created_at)` — active filtering

### pincode_case_audit_log
```
ID, PricingID (→ pincode_case_pricing._id), CaseSKU, Zone, Action ("CREATE"/"UPDATE"),
ChangedByEmail, ChangedAt, FieldsChanged []string, NewValues map[string]any, Platform
```

**Index:** `(pricing_id, changed_at -1)`

---

## API Routes (`/pincode-cases`, JWKS auth)

| Method | Path | Handler | Notes |
|--------|------|---------|-------|
| POST | `/` | CreatePincodeCasePricing | Validates case exists & available, pricing math, unique pincodes |
| GET | `/` | GetPincodeCasePricing | Paginated, filters: case_sku, zone, pincode, is_active |
| GET | `/:id` | GetPincodeCasePricingByID | |
| PUT | `/` | UpdatePincodeCasePricing | By `?id=` or `?case_sku=&zone=`, tracks changed fields |
| GET | `/:id/audit-logs` | GetAuditLogs | Sorted newest first |
| GET | `/check-pincode` | CheckPincodeAvailability | `?sku=&pincode=` — check if pincode already mapped |
| GET | `/cases-summary` | GetCasesWithZoneCount | Aggregation: cases + zone count, pagination |
| GET | `/zone-pricing` | GetZonePricingDetails | `?case_sku=&zone=` — full pricing breakdown with calculated case_price |
| GET | `/distinct/zones` | GetDistinctPincodeCaseZones | Unique zones list |

**Route file:** `routes/pincodeCaseRoutes.go`
**Registered in:** `routes/route.go`

---

## Pricing Validation Logic

On create/update, fetches case details from `cases` collection:
- `case_qty` = case.quantity
- `variant_value` = parsed from case.variant (e.g., "500 ml" → 0.5 if base UOM is "l")

**Price checks (±0.01 tolerance):**
- `unit_price = unit_case_price / case_qty`
- `uom_unit_price = unit_case_price / (case_qty * variant_value)`

**Default tier:** 2nd pricing tier (or 1st if only one) gets `default: true`

**UOM normalization:** ltr/liter/litre→l, gram/g→gm, kilo/kilogram→kg, ml stays ml

---

## Side Effects
- **Create:** Sets `has_pincode_case: true` on the case document in `cases` collection
- **Create/Update:** Async audit log entry via goroutine
- **Duplicate pincode:** Returns 409 Conflict (MongoDB error code 11000)

---

## Consumer: Inventory-Management-Backend Search Product API

**Route:** `GET /cases/search/product?pincode=XXXXX&...`
**File:** `Inventory-Management-Backend/routes/cases/controllers/cases-controller.js`

**Integration type:** Direct MongoDB query (NOT HTTP call to utilities-go)

```javascript
// Uses generic Mongoose schema (strict: false) for pincode_case_pricing
const query = {
  is_active: true,
  pincodes_array: { $in: [retailer_pincode] },
  case_sku: case_id,
};
pincode_case_pricing.findOne(query, { pricing: 1 }).lean();
```

**Pincode source:** `?pincode=` query param, or looked up from `Retailer.address.pincode` via retailer phone

**Pricing priority:**
1. Pincode case pricing (if found for retailer's pincode + case_sku)
2. Default case pricing (from `cases` collection)

**Price formatting differs:** When pincode pricing found, formats `per_unit_price` as `{unit_price}/{variant}` instead of UOM-based calculation.

---

## Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `handlers/pincode_cases.go` | All 9 handlers + 9 helpers | ~1330 |
| `models/pincode_case_model.go` | Models, request/response structs | ~90 |
| `routes/pincodeCaseRoutes.go` | Route registration | ~40 |
| `scripts/test-pincode-api.sh` | curl-based API tests (12 cases) | — |
| `docs/pincode_case_pricing_docs/` | API docs + quickstart | — |

---

## Business Rules
1. Pincode can't exist in multiple zones for same case_sku (unique index)
2. Case must exist with `is_available: true` and `quantity > 0`
3. No hard delete — use `is_active: false` for soft deactivation
4. All pricing tiers must be mathematically consistent
5. Audit trail is immutable (no edit/delete of logs)
