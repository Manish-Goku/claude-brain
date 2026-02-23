# Deal Module — API Request/Response Contract

Base path: `/deal` (mounted via auto-discovery in `routes/deal/route.cjs`)

---

## POST /deal/create/:contactId
**Create a new deal from a contact**

### Request
- Params: `contactId` (e.g. `"C-1234"`)
- Body (optional overrides):
```json
{
  "requirements": [
    {
      "productSku": "SKU-001",
      "productName": "Neem Oil",
      "productPrice": 500,
      "productQuantity": 10,
      "case_qty": null,
      "gst": "18%",
      "totalAmount": 5000,
      "item_type": "product"
    }
  ],
  "dealNature": "Hot",
  "estimatedValue": 50000,
  "estimatedClosureDate": "2025-03-15",
  "notes": ["Initial contact made"]
}
```

### Response (201)
```json
{
  "message": "Deal created successfully.",
  "deals": {
    "dealId": "DL-1001",
    "contactId": "C-1234",
    "firstName": "john",
    "lastName": "doe",
    "contact": "9876543210",
    "deal_owner": { "agentId": "A0-1001", "agentRef": "...", "katyayani_id": "..." },
    "current_stage": "Open",
    "requirements": [...],
    "priority": 2,
    "dealSubType": "Other Leads",
    "pipeline_stage_history": [{ "stage": "Open (contacting)", "time": "..." }],
    "deal_owner_history": ["manish.pandey"],
    "created_by": "manish.pandey",
    /* all common fields inherited from contact */
  }
}
```

**Side effects**:
- Contact's `current_stage` set to `{ stage_name: "won", sub_stage: 0 }`
- Contact's `dealIds` gets new dealId pushed
- Pipeline updated to `won: true`
- Deal pipeline doc created

**Validations**:
- Contact must have `farmer_profile`
- Contact stage cannot be `close` or `lost`

---

## GET /deal/:dealId
**Get single deal by dealId**

### Request
- Params: `dealId` (e.g. `"DL-1001"`)

### Response (200)
```json
{
  "message": "Deal fetched successfully",
  "deals": {
    "dealId": "DL-1001",
    "contactId": "C-1234",
    "firstName": "...",
    "deal_owner": { ... },
    "current_stage": "Estimate",
    "requirements": [...],
    "priority": 2,
    "company_pricing": [...],
    "notes": [...],
    "deal_lost_reason": null,
    "follow_up_time": null,
    "pipeline_stage_history": [...],
    "quotationIds": [...],
    "specialDealIds": [...],
    "order_id": null,
    /* all deal fields except updated_data */
  }
}
```
- `403` if agent not authorized (non-admin, non-owner)

---

## GET /deal/all
**Get all deals (paginated)**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1, limit fixed at 50 |

### Response (200)
```json
{
  "success": true,
  "deals": [ /* deal documents, sorted by priority ASC */ ],
  "hasNextPage": true
}
```

---

## GET /deal/search
**Search/filter deals with pagination**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1 |
| limit | number | Max 50 |
| range | number | Days ahead for date range (default 1) |
| firstName | string | Regex prefix (lowercased) |
| lastName | string | Regex prefix (lowercased) |
| email | string | Regex prefix (lowercased) |
| contact | string | Exact (10 digit) or suffix regex |
| contact2 | string | Same as contact |
| State | string | Uppercased |
| country | string | Uppercased |
| city | string | Uppercased |
| customerId | string | Resolves to contact number |
| current_stage | string | Pipeline stage filter |
| deal_owner.agentId | string | Filter by agent |
| dealId | string | Direct match |
| contactId | string | Direct match |
| latestActivityOn | string | Date range filter |
| updatedAt | string | Date range filter |
| createdAt | string | Date range filter |
| priority | number | Numeric filter |
| estimatedValue | number | Numeric filter |
| deal_source | string | Maps to contactSources |
| dealSubType | string | Sub type filter |
| hasSpecialDeal | string | `"true"` / `"false"` filters by existence of special deals |

### Response (200)
```json
{
  "success": true,
  "deals": [ /* deal documents with updated_data: [] */ ],
  "hasNextPage": false,
  "total_count": 25
}
```

---

## GET /deal/latest-deal
**Get best matching deal for order creation**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| contactId | string | Required |
| product_skus | string[] | Array of SKUs to match against requirements |

### Response (200)
```json
{
  "success": true,
  "deals": [ /* 0 or 1 best matching deal */ ]
}
```
Filters: Excludes `Won Deal`, `Closed Deal`, `Deal Lost` stages. Matches by `deal_owner.agentId = req.agentId` and overlapping product SKUs in requirements.

---

## PUT /deal/:dealId
**Update a deal**

### Request
- Params: `dealId`
- Body: Any deal fields to update. Special behaviors:

#### Pipeline Update
```json
{
  "pipelineData": { /* pipeline update object for updatePipeline helper */ },
  "deal_lost_reason": { "reason": { "primary": "Price too high" }, "remarks": "..." }
}
```

#### Requirements Update
```json
{
  "requirements": [
    { "productSku": "...", "productName": "...", "productPrice": 500, "productQuantity": 10, "totalAmount": 5000 }
  ]
}
```

#### Owner Update
```json
{
  "deal_owner": { "agentId": "A0-1002", "agentRef": "...", "katyayani_id": "..." }
}
```

#### Follow-up
```json
{
  "follow_up_time": "2025-02-20T10:00:00Z"
}
```

#### Notes (allowed even in Won/Closed stage)
```json
{
  "notes": "Follow up call completed"
}
```

**Restrictions**: When deal is in `Won Deal` or `Closed Deal`, only `notes` can be added.

### Response (200)
```json
{
  "success": true,
  "message": "Deal updated successfully",
  "deals": { /* updated deal without updated_data */ }
}
```

**Side effects**:
- Pipeline change also updates `description` and `pipeline_stage_history`
- `order_id` update marks all quotations as expired
- Agent dashboard updated on stage change
- Cron bin scheduled for follow-up time

---

## Deal Pipeline Stages
| Stage | Description |
|---|---|
| Open | Just created, no activity |
| Estimate | Send price estimate/quotation |
| Negotiation | Price negotiation in progress |
| Company Deal Negotiation | Company-level negotiation |
| Company Deal Approval | Pending company approval |
| Deal Follow-Up 1 | Following up after estimate |
| Dormant (Deal Follow-Up 2) | Long-term follow-up (waiting for tender etc.) |
| Won Deal | Successfully closed |
| Closed Deal | Customer decided not to order |
| Deal Lost | Deal lost |

---

## Deal Requirements Schema
```json
{
  "productSku": "string (required)",
  "productName": "string (required)",
  "productPrice": "number >= 0 (required)",
  "productQuantity": "number >= 1 (required)",
  "case_qty": "number >= 0 (optional)",
  "gst": "string (optional)",
  "totalAmount": "number >= 0 (required)",
  "item_type": "enum: product | combo | case"
}
```
