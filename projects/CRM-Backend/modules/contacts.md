# Contact Module — API Request/Response Contract

Base path: `/contact` (mounted via auto-discovery in `routes/contact/route.cjs`)

---

## GET /contact/all
**Get all active contacts (non-terminal pipeline stages)**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1, limit fixed at 50 |

### Response (200)
```json
{
  "success": true,
  "contacts": [
    {
      "contactId": "C-1234",
      "leadId": "K0-5678",
      "firstName": "john",
      "lastName": "doe",
      "contact": "9876543210",
      "email": "...",
      "contact_owner": { "agentId": "A0-1001", "agentRef": "...", "katyayani_id": "..." },
      "current_stage": { "stage_name": "contacting", "sub_stage": 1 },
      "dealIds": ["DL-100"],
      "quotationIds": ["QT-200"],
      "priority": 7,
      "State": "MAHARASHTRA",
      "city": "pune",
      "status": "Active",
      "contactSubType": "Other Leads",
      "pipeline_stage_history": [{ "stage": "Open (contacting)", "time": "..." }],
      "farm_details": { ... },
      "tags": [...],
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "hasNextPage": true
}
```
Filters: `current_stage.stage_name NOT IN [dormant, lost, won, lead_closed, lead_unqualified]`, `priority != 0`

---

## GET /contact/:contactId
**Get single contact by contactId**

### Request
- Params: `contactId` (e.g. `"C-1234"`)

### Response (200)
```json
{
  "success": true,
  "contacts": [
    { /* full contact document except updated_data */ }
  ]
}
```
- Returns `404` if not found
- Returns `403` if agent not authorized (non-admin, non-owner)

---

## GET /contact/search
**Search/filter contacts with pagination**

### Request (Query Params)
| Param | Type | Notes |
|---|---|---|
| page | number | Default 1 |
| limit | number | Only for admins |
| range | number | Days ahead for date range (default 1) |
| firstName | string | Regex prefix (lowercased) |
| lastName | string | Regex prefix (lowercased) |
| email | string | Regex prefix (lowercased) |
| contact | string | Exact (10 digit) or suffix regex |
| State | string | Uppercased |
| country | string | Uppercased |
| city | string | Uppercased |
| customerId | string | Resolves to contact number |
| current_stage.stage_name | string | Pipeline stage filter |
| current_stage.sub_stage | number | Sub-stage filter |
| priority | number | Numeric filter, auto-excludes 0 |
| hasDeal | string | `"true"` filters contacts with dealIds |
| source | string | Maps to `contactSources $in` |
| tags | string | Comma-separated tag names |
| assigned_date | string | Date range filter |
| created_at | string | Date range filter |
| updatedAt | string | Date range filter |
| createdAt | string | Date range filter |
| priorityUpdateTime | string | Date range filter |
| contact_owner.agentId | string | Filter by agent |
| contactSubType | string | Sub type filter |
| `*_option` | string | Inclusion/exclusion filter modifier |

Auto-filters: `priority != 0`. If no stage/contact/email specified, excludes `lead_unqualified`.

### Response (200)
```json
{
  "success": true,
  "contacts": [
    {
      "firstName": "...",
      "lastName": "...",
      "gst_number": "...",
      "contact": "...",
      "email": "...",
      "leadId": "...",
      "contactSubType": "...",
      "status": "...",
      "contactId": "...",
      "current_stage": { "stage_name": "...", "sub_stage": 0 },
      "address1": "...",
      "address2": "...",
      "city": "...",
      "State": "...",
      "country": "...",
      "createdAt": "...",
      "updatedAt": "...",
      "priority": 7,
      "contact_owner": { ... },
      "pinCode": "...",
      "contactSources": [...],
      "farm_details": { ... },
      "tags": [{ "name": "..." }]
    }
  ],
  "hasNextPage": false,
  "count": 25,
  "totalcount": 25
}
```

---

## GET /contact/dashboard
**Get contact counts by stage for the logged-in agent**

### Response (200)
```json
{
  "success": true,
  "all_leads": 500,
  "active_leads": 200,
  "inactive_leads": 100,
  "no_status_leads": 200,
  "contacting1_leads": 50,
  "contacting2_leads": 30,
  "contacted1_leads": 40,
  "contacted2_leads": 20,
  "won_leads": 60,
  "lost_leads": 30,
  "dormant_leads": 20,
  "closed_leads": 10,
  "open_leads": 40
}
```

---

## POST /contact/create
**Create or update a contact**

### Request Body
```json
{
  "contact": "9876543210",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "contact_owner": { "agentId": "A0-1001", "agentRef": "...", "katyayani_id": "..." },
  "contactSources": ["CRM", "Shopify"],
  "current_stage": { "stage_name": "contacting", "sub_stage": 1 },
  "contactSubType": "Other Leads",
  "tags": ["tag1", "tag2"],
  "address1": "...",
  "city": "...",
  "State": "...",
  "country": "...",
  "pinCode": "..."
}
```
**Required**: `contact`

### Response (201) — New contact
```json
{
  "success": true,
  "contacts": { /* saved contact document */ }
}
```

### Response (200) — Existing contact updated
```json
{
  "Success": true,
  "message": "Contact updated",
  "contact": { /* updated contact document */ }
}
```
**Side effects**: Creates Pipeline document for new contacts.

---

## PUT /contact/:contactId
**Update a contact**

### Request
- Params: `contactId`
- Body: Any contact fields to update. Special behaviors:
  - `priorityUpdateTime` — Sets priority to 15, schedules cron bin, returns early
  - `Pipeline` — Updates pipeline stage (calls `updatePipeline` helper). Cannot update from `lead_unqualified/won/lead_closed` stages
  - `Pipeline.unqualification_reason` — Set when stage becomes `lead_unqualified`
  - `notes` (string) — Pushed to notes array
  - `query` (array) — Validated and pushed to query array
  - `tags` (array of strings) — Created in Tags collection, pushed to tags array
  - `farmer_profile` — Also updates customer profile

### Response (200)
```json
{
  "success": true,
  "message": "Contact updated successfully",
  "contacts": { /* updated contact without updated_data */ }
}
```

---

## PUT /contact/csvcontacts
**Update contacts via CSV upload**

### Request
- Content-Type: `multipart/form-data`
- Field: `file` (CSV)
- Required CSV column: `contactId`

### Response (200)
```json
{
  "success": true,
  "message": "Successfully updated 50 contacts",
  "updatedContacts": ["C-1234", "C-1235"],
  "invalidRows": []
}
```

---

## POST /contact/csvcontacts
**Create contacts via CSV upload**

### Request
- Content-Type: `multipart/form-data`
- Field: `file` (CSV)
- Required CSV columns: `firstName`, `contact`

### Response (200)
```json
{
  "success": true,
  "message": "50 leads processed successfully",
  "invalidRows": []
}
```

---

## DELETE /contact/:contactId
**Delete a contact and its pipeline**

### Request
- Params: `contactId`

### Response (200)
```json
{
  "success": true,
  "contacts": { /* deleted contact */ },
  "pipelines": { /* deleted pipeline */ }
}
```

---

## PATCH /contact/verify-licence
**Verify licence info for a contact**

### Request Body
```json
{
  "contactId": "C-1234",
  "verify_licence": true,
  "valid_upto": "2025-12-31"
}
```

### Response (200)
```json
{
  "success": true,
  "message": "licence has been verified successfully for lead with contact id as : C-1234"
}
```

---

## PATCH /contact/transfer-contacts
**Transfer contacts between agents** (Super Admin / Admin only)

### Request
- Body:
```json
{
  "transfer_from": "A0-1001",
  "transfer_to": "A0-1002"
}
```
- Query: Same search params as `/contact/search` + `limit`

### Response (200)
```json
{
  "success": true,
  "message": "150 leads were transfered"
}
```

**Side effects**: Calls `transfer_everything` which updates leads, contacts, deals, quotations.

---

## Pipeline Stages (contact)
| Stage | Description |
|---|---|
| Open (contacting) | Initial open state |
| contacting (sub 1,2) | Follow-up with not-connected customers |
| contacted1 | Gathered info, provided solutions |
| contacted2 | Customer needs longer duration (10-15 days) |
| dormant | Similar to contacted2, long-term follow-up |
| lost | Did not convert |
| won | Successfully converted |
| lead_closed | Closed without conversion |
| lead_unqualified | Does not meet qualification criteria |
