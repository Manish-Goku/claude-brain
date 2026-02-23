# Lead Module — API Request/Response Contract

Base path: `/lead` (mounted via auto-discovery in `routes/lead/route.cjs`)

---

## GET /lead/search
**Search/filter leads with pagination**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1 |
| limit | number | Max 50 for non-admins |
| tags | string | Comma-separated tag names |
| startDate | string | ISO date, filters `assigned_Date >= startDate` |
| endDate | string | ISO date, filters `assigned_Date <= endDate` |
| order | string | `"true"` to filter leads with order_history |
| range | number | Days ahead for date range queries (default 1) |
| firstName | string | Regex prefix match (lowercased) |
| lastName | string | Regex prefix match (lowercased) |
| contact | string | Exact (10 digit), suffix regex, or last 10 chars |
| State | string | Uppercased |
| country | string | Uppercased |
| customerId | string | Resolves to contact number via Customer collection |
| dispossession_status | string | `"true"` / `"false"` -> boolean |
| leadStatus | string | `"Fresh Lead"` / `"Existing Lead"` |
| leadSubType | string | e.g. `"Other Leads"`, `"Franchise"` |
| priority | number | Supports range `priority[gte]=5&priority[lte]=10` |
| prospect_value | number | Supports range queries |
| platforms | string | Comma-separated, `$in` filter |
| isCustomer | string | `"true"` / `"false"` |
| `*_option` | string | Inclusion/exclusion filter modifier |

Non-admins: auto-filtered to `leadOwner.agentId = req.agentId` and `is_enabled = true`.

### Response (200)
```json
{
  "success": true,
  "total": 150,
  "page": 3,
  "limit": 50,
  "data": [
    {
      "leadId": "K0-1234",
      "firstName": "john",
      "lastName": "doe",
      "contact": "9876543210",
      "email": "...",
      "leadOwner": {
        "agentId": "A0-1001",
        "agentRef": { "firstname": "...", "lastname": "...", "email": "...", "contact": "...", "state": "...", "address": "...", "city": "...", "country": "..." }
      },
      "tags": [{ "name": "tag1" }],
      "leadStatus": "Fresh Lead",
      "leadSubType": "Other Leads",
      "priority": 5,
      "State": "MAHARASHTRA",
      "city": "pune",
      "contact2": "+919876543210",
      "dispossession_status": false,
      "callStatus": { "status": "Not Performed", "callTime": "...", "nextPriorityUpdate": null },
      "farm_details": { ... },
      "assigned_Date": "...",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

---

## GET /lead/all/:leadId
**Get single lead by leadId**

### Request
- Params: `leadId` (string, e.g. `"K0-1234"`)

### Response (200)
```json
{
  "success": true,
  "message": "Lead retrieved successfully",
  "data": {
    "leadId": "K0-1234",
    "firstName": "...",
    "lastName": "...",
    "contact": "...",
    "leadOwner": { "agentId": "...", "agentRef": { /* populated agent */ } },
    "tags": [{ "name": "..." }],
    "farm_details": { "Crop_name": [{ "cropRef": { /* populated crop */ } }] },
    /* all lead fields except _id, __v */
  }
}
```
- Returns `201` with empty body if lead not found
- Returns `403` if agent not authorized

---

## GET /lead/leadOwner
**Get lead owner update history (aggregated)**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1 |
| pageSize | number | Default 100 |

### Response (200)
```json
{
  "success": true,
  "data": [
    {
      "_id": "...",
      "latestUpdate": { "updatedBy": "...", "updatedFields": { "leadOwner": { ... } } }
    }
  ],
  "pagination": { "currentPage": 1, "pageSize": 100, "totalPages": 5, "totalDocuments": 450 }
}
```

---

## GET /lead/socket-event
**Emit socket event for call-related lead lookup**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| email | string | Agent email |
| phone_number | string | Lead contact |
| call_status | string | Call status |

### Response (200)
```json
{
  "success": true,
  "message": "socket emission has been done the respective agent or admin"
}
```

---

## GET /lead/call-leads
**Get call leads (paginated)**

### Request (Query Params)
| Param | Type |
|---|---|
| page | number (default 1) |

### Response (200)
```json
{
  "success": true,
  "leads": [ /* call_leads documents */ ],
  "total_pages": 5,
  "current_page": 1
}
```

---

## GET /lead/search/call-leads
**Search call leads**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1 |
| limit | number | Max 50 |
| normal_lead_created | string | `"true"` / `"false"` |
| agentId | string | Maps to `leadOwner.agentId` |
| contact | string | Exact or suffix match |
| priority | number | Supports range with `_operator`, `_range` |
| duration | number | Supports range |

### Response (200)
```json
{
  "success": true,
  "leads": [ ... ],
  "total_pages": 3,
  "current_page": 1
}
```

---

## POST /lead/create
**Create a new lead**

### Request Body
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "contact": "9876543210",
  "email": "john@example.com",
  "leadStatus": "Fresh Lead",
  "leadSubType": "Other Leads",
  "leadOwner": { "agentId": "A0-1001", "agentRef": "mongoObjectId" },
  "tags": ["tag1", "tag2"],
  "State": "MAHARASHTRA",
  "city": "Pune",
  "country": "INDIA",
  "pinCode": "411001",
  "address1": "...",
  "requirements": "...",
  "farm_details": { ... },
  "source": { "sourceId": "...", "sourceRef": "..." },
  "owner_history": ["A0-1001"]
}
```
**Required**: `firstName`, `contact`

### Response (201) — Lead created
```json
{
  "success": true,
  "message": "Lead created successfully",
  "lead": { /* full lead document */ }
}
```

### Response (200) — RLMS redirect (not created in CRM)
```json
{
  "success": false,
  "message": "Lead already exists in RLMS. Not creating in CRM/SA.",
  "assignedDepartment": "RLMS",
  "leadSource": "RLMS"
}
```

### Response (400)
```json
{
  "success": false,
  "message": "Can't create the lead as it already exists in the DB with leadId as K0-1234"
}
```

---

## PUT /lead/:leadId
**Update a single lead**

### Request
- Params: `leadId`
- Body: Any lead fields to update. Special behaviors:
  - `query` — Triggers `QueryUpdate` helper and returns early
  - `dispossession` — Triggers `dispossession_update` helper and returns early
  - `user_message_time` — Also updates matching contact
  - `callStatus.status` — Updates priority, cron bin, callStatusHistory
  - `tags` (array of strings) — Created in Tags collection, added via `$addToSet`
  - `notes` (string) — Appended to existing notes array
  - `order_history` (array) — Merged with existing
  - `leadProfile` — Also updates customer profile

### Response (200)
```json
{
  "success": true,
  "message": "Leads updated successfully",
  "data": [ /* single updated lead with populated leadOwner and farm_details */ ]
}
```

---

## POST /lead/update-multiple
**Bulk assign leads to a new owner**

### Request
- Query: `assign_type` — `"A"` (assign only), `"R"` (retention), `"RA"` (retention+acquisition, also creates contacts)
- Body:
```json
{
  "leadIds": ["K0-1234", "K0-1235"],
  "leadOwner": { "agentId": "A0-1001", "agentRef": "mongoObjectId" }
}
```

### Response (200)
```json
{
  "success": true,
  "message": "Leads updated successfully",
  "data": [ /* updated leads with populated leadOwner */ ]
}
```

**Side effects**: Socket emission, contact creation (if RA/R), cron bin scheduling, interakt assignment, sync leads with contacts.

---

## POST /lead/csvleads
**Create leads via CSV upload**

### Request
- Content-Type: `multipart/form-data`
- Field: `file` (CSV with headers matching Lead schema fields)
- Required CSV columns: `firstName`, `contact`

### Response (200)
```json
{
  "success": true,
  "message": "150 leads processed successfully",
  "invalidRows": [{ "contact": "...", "error": "..." }]
}
```

---

## POST /lead/assign/by/csv
**Assign leads via CSV** (Super Admin / Admin only)

### Request
- Content-Type: `multipart/form-data`
- Field: `file` (CSV)
- Required CSV columns: `lead_id`, `katyayani_id`
- Optional: `assign_type` (`A`, `R`, `RA`)

### Response (200)
```json
{
  "success": true,
  "errors": [],
  "invalid_rows": []
}
```

---

## POST /lead/assign-call-leads
**Assign call leads to an agent (moves from call_leads to Leads collection)**

### Request
- Query: `assign_type` — `"RA"`, `"R"`, or null (assign only)
- Body:
```json
{
  "leadIds": ["CL-1234"],
  "agentId": "A0-1001"
}
```

### Response (200)
```json
{
  "success": true,
  "message": "All leads have been successfully assigned to agent A0-1001"
}
```

---

## POST /lead/create-interact-lead
**Create lead in InteractBin collection**

### Request Body
```json
{
  "firstName": "...",
  "contact": "9876543210",
  "leadSubtype": "...",
  "interakt_sub_source": "...",
  "State": "...",
  "city": "...",
  "country": "...",
  "pinCode": "...",
  "address1": "...",
  "requirements": "..."
}
```

### Response (201)
```json
{
  "success": true,
  "message": "Interact Lead created successfully",
  "data": { /* InteractBin document */ }
}
```

---

## POST /lead/statetask
**Create state lookup tasks for contacts**

### Request Body
```json
{ "contacts": ["9876543210", "9876543211"] }
```

### Response (200)
```json
{
  "success": true,
  "leadIds": ["K0-1234"],
  "message": "State Tasks were successfully created."
}
```

---

## POST /lead/create/contacts
**Batch create contacts for all leads of a given agent**

### Request Body
```json
{ "katyayani_id": "manish.pandey" }
```
- Query: Same search params as `/lead/search` (tags, etc.)

### Response (201)
```json
{
  "success": true,
  "message": "All contacts have been created successfully"
}
```

---

## POST /lead/call-event
**Emit socket event for a call event**

### Request Body
```json
{
  "phone_number": "9876543210",
  "email": "agent@example.com",
  "call_type": "inbound",
  "event_type": "call-started",
  "timestamp": "2025-01-01T10:00:00Z"
}
```

### Response (200)
```json
{
  "call_type": "inbound",
  "email": "...",
  "event_type": "call-started",
  "lead_id": "K0-1234",
  "phone_number": "9876543210",
  "timestamp": "..."
}
```

---

## PATCH /lead/transfer-leads
**Transfer leads between agents** (Super Admin / Admin only)

### Request
- Body:
```json
{
  "transfer_from": "A0-1001",
  "transfer_to": "A0-1002"
}
```
- Query: Same search params as `/lead/search` + `limit`

### Response (200)
```json
{
  "success": true,
  "message": "150 leads were transfered"
}
```

**Side effects**: Calls `transfer_everything` which updates leads, contacts, deals, quotations.

---

## PATCH /lead/assign-retailers
**Upsert leads as retailers in Retailer collection**

### Request Body
```json
{
  "leadIds": ["K0-1234"],
  "agentId": "A0-1001"
}
```

### Response (201)
```json
{
  "success": true,
  "message": "Inserted 5 retailers and updated 3 retailers' owner email"
}
```

---

## PATCH /lead/is-enabled
**Enable/disable leads and their contacts** (Super Admin / Admin only)

### Request Body
```json
{
  "contacts": ["9876543210", "9876543211"],
  "option": true
}
```

### Response (200)
```json
{
  "success": true,
  "message": "All the leads and contacts selected have been enabled for their owners"
}
```

---

## PATCH /lead/update-interakt-time/:leadId
**Send interakt message and update agent reply time**

### Request
- Params: `leadId`
- Body:
```json
{
  "fullPhoneNumber": "+919876543210",
  "message": "Hello from CRM"
}
```

### Response (200)
```json
{
  "success": true,
  "message": "Successfully sent the message to the interakt client with crm lead id as : K0-1234"
}
```

---

## PATCH /lead/assign
**Round-robin assign leads from unassigned pool to agents** (Super Admin / Admin / Floor Manager)

### Request Body
```json
{
  "state": "MAHARASHTRA",
  "agent_ids": ["A0-1001", "A0-1002"],
  "limit": 50
}
```

### Response (200)
```json
{
  "success": true,
  "assigned_leads": {
    "A0-1001": ["K0-1234", "K0-1236"],
    "A0-1002": ["K0-1235", "K0-1237"]
  }
}
```

---

## DELETE /lead/interakt/:contact
**Delete an interakt lead from InteractBin**

### Request
- Params: `contact` (phone number)

### Response (200)
```json
{
  "success": true,
  "deletedLead": { /* deleted InteractBin document */ }
}
```
