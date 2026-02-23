# Quotation Module — API Request/Response Contract

Base path: `/quotation` (mounted via auto-discovery in `routes/quotation/route.cjs`)

---

## POST /quotation/
**Create a new quotation**

### Request
- Query Params:
  - `deal_id` (optional) — If provided, inherits common fields from the deal
  - `channel_name` (optional) — Used when creating without a deal

- Body:
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "contact": "9876543210",
  "email": "...",
  "line_item": [
    {
      "item": "Neem Oil 1L",
      "qty": 10,
      "case_qty": null,
      "singleItemPrice": 500,
      "sku": "SKU-001",
      "gst": "18%",
      "total_amount": 5000,
      "item_type": "product"
    }
  ],
  "expiry_date": "2025-02-20",
  "status": "Created",
  "total_amount": 5000,
  "prepaid_amount": 2000,
  "cod_amount": 3000,
  "with_utr": true,
  "already_collected_amount": 1000,
  "utr_infos": [{ "bank_name": "HDFC", "amount": 1000, "utr_number": "UTR123" }],
  "prepaid_percentage": 40,
  "payment_terms": "50% advance",
  "payment_type": "partial",
  "subject": "Quote for Neem Oil",
  "terms_and_condition": "...",
  "shipping_address": { "address1": "...", "city": "...", "state": "...", "country": "...", "pinCode": "..." },
  "billing_address": { "address1": "...", "city": "...", "state": "...", "country": "...", "pinCode": "..." },
  "place_of_supply": "Maharashtra",
  "gst_number": "...",
  "reference": "...",
  "notes": "...",
  "min_advance_percentage": 30
}
```

### Response (200)
```json
{
  "success": true,
  "quotations": {
    "quotationId": "QT-1001",
    "firstName": "...",
    "contact": "...",
    "associated_dealid": "DL-1001",
    "associated_contactid": "C-1234",
    "quotation_owner": { "agentId": "A0-1001", "agentRef": "...", "katyayani_id": "..." },
    "line_item": [...],
    "status": "Created",
    "expiry_date": "...",
    "total_amount": 5000,
    "prepaid_amount": 2000,
    "cod_amount": 3000,
    "amount_to_be_collected": 1000,
    "already_collected_amount": 1000,
    "with_utr": true,
    "expired": false,
    "order_id": null,
    "channel": null,
    "user_approved": null,
    /* ... all fields */
  }
}
```

**Side effects**:
- If `deal_id` provided: pushes `quotationId` to both Deal and Contact
- If `status == "Created"`: triggers module events (Slack notification via event system)

**Amount Auto-Calculations**:
- `with_utr == true`: `amount_to_be_collected = prepaid_amount - already_collected_amount`, `cod_amount = total_amount - prepaid_amount`
- `with_utr == false`: `prepaid_amount = total_amount - cod_amount`

---

## GET /quotation/:quotationId
**Get single quotation**

### Request
- Params: `quotationId` (e.g. `"QT-1001"`)

### Response (200)
```json
{
  "success": true,
  "quotations": { /* full quotation except updated_data */ }
}
```
- `403` if not found or agent not authorized

---

## GET /quotation/all
**Get all quotations (paginated)**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1, limit fixed at 50 |

### Response (200)
```json
{
  "success": true,
  "quotations": [ /* quotation documents sorted by createdAt DESC */ ],
  "hasNextPage": true
}
```

---

## GET /quotation/search
**Search/filter quotations**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1, limit fixed at 50 |
| contact | string | Exact (10 digit) or suffix regex |
| contact2 | string | Same as contact |
| State | string | Uppercased |
| country | string | Uppercased |
| city | string | Uppercased |
| associated_dealid | string | Uppercased |
| associated_contactid | string | Uppercased |
| quotation_owner.agentId | string | Filter by agent |
| quotationId | string | Direct match |
| status | string | `"Draft"` / `"Created"` |
| expired | string | `"true"` / `"false"` -> boolean |
| user_viewed | string | `"true"` / `"false"` -> boolean |
| order_attempted | string | `"true"` / `"false"` -> boolean |
| prepaid_amount | number | Supports range with `_operator`, `_range` |
| prepaid_percentage | number | Supports range |
| cod_amount | number | Supports range |
| expiry_date | string | Date range |
| created_at | string | Date range |
| updatedAt | string | Date range |
| createdAt | string | Date range |
| user_viewed_at | string | Date range |
| order_id | string | Direct match |
| channel | string | Direct match |
| user_approved | string | `"approved"` / `"rejected"` / null |

### Response (200)
```json
{
  "success": true,
  "quotations": [ /* matched quotation documents */ ],
  "hasNextPage": false
}
```

---

## GET /quotation/check/validity/:quotationId
**Check if a quotation is still valid for order placement**

### Request
- Params: `quotationId`
- Query: `contact` (string, required)

### Response (200)
```json
{
  "valid_quotation": true,
  "order_placed": false,
  "order_id": null
}
```
Validity checks:
1. Quotation exists with matching contact
2. Not expired (`expiry_date > now`)
3. Associated deal not in `Won Deal` or `Closed Deal`

---

## PUT /quotation/:quotationId
**Update a quotation**

### Request
- Params: `quotationId`
- Body: Any quotation fields to update. Special behaviors:
  - `line_item` — Must be array, replaces existing
  - `associated_dealid` / `associated_contactid` — Blocked from update
  - Amount auto-calculations same as create
  - Cannot update expired quotation
  - Cannot update if associated deal has `order_id`

```json
{
  "line_item": [{ "item": "...", "qty": 5, "singleItemPrice": 300, "sku": "...", "total_amount": 1500 }],
  "total_amount": 1500,
  "prepaid_amount": 500,
  "cod_amount": 1000,
  "status": "Created",
  "order_id": "ORD-5001"
}
```

### Response (200)
```json
{
  "success": true,
  "quotations": { /* updated quotation */ }
}
```

**Side effects**:
- `status == "Created"` -> calls `update_deal_stage` (moves deal pipeline to Estimate)
- `order_id` set -> calls `update_deal_stage(dealId, "Won Deal")`
- Triggers module events for Slack

---

## POST /quotation/send/to/interact
**Send quotation via Interakt**

### Request Body
```json
{
  "quotation": { /* quotation object to send */ }
}
```

### Response
- `204` No Content on success

---

## POST /quotation/correct-quotation-owner
**Sync quotation owners with their deal owners (admin utility)**

### Response (200)
```json
{
  "success": true,
  "message": "Quotation owners corrected successfully",
  "stats": {
    "total": 500,
    "updated": 50,
    "already_correct": 440,
    "deal_not_found": 10
  }
}
```

---

## Line Item Schema
```json
{
  "item": "string (product name)",
  "qty": "number",
  "case_qty": "number (optional)",
  "singleItemPrice": "number",
  "sku": "string (product SKU)",
  "gst": "string (optional)",
  "total_amount": "number",
  "item_type": "enum: product | combo | case | promotional"
}
```

---

## Quotation Status Enum
- `Draft` — Work in progress
- `Created` — Finalized and ready

## Amount Validation Rules (when `with_utr == true`)
1. `already_collected_amount + amount_to_be_collected = prepaid_amount` (1 rupee tolerance)
2. `amount_to_be_collected <= prepaid_amount`
3. `already_collected_amount <= prepaid_amount`
4. `sum(utr_infos[].amount) = already_collected_amount` (1 rupee tolerance)
