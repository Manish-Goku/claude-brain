# CRM Backend Quick Reference

**Last Updated**: February 14, 2026

## Project Overview

Node.js/Express CRM backend with MongoDB, featuring normalized schemas and TypeScript migration.

## Key Technologies

- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: MongoDB with Mongoose
- **Language**: TypeScript + CommonJS (migration in progress)
- **Auth**: JWT-based authentication
- **Caching**: Redis

## Schema Normalization (In Progress — branch: schema_normalization)

Database schema normalization with TypeScript migration. 9 core endpoints migrated to V2, 13 legacy endpoints still on old controller.

See [schema-normalization.md](./modules/schema-normalization.md) for full details.

### Active Helper Utilities

```typescript
// PII operations (ACTIVE)
import { handlePII, normalizePhoneNumber } from './routes/lead/utils/piiHandler';
import { handleAddress } from './routes/lead/utils/addressHandler';
import { createQuery, getLatestQuery } from './routes/lead/utils/queryHandler';
import { createNote, getNoteDescriptions } from './routes/lead/utils/notesHandler';
import { mapLeadToOldFormat, mapRequestToNewSchema } from './routes/lead/utils/schemaMapper';

// Search (uses CJS helper, NOT searchHelperV2.ts)
const { build_search_query_ts } = require('./routes/lead/utils/searchHelpers.cjs');

// Reference only (NOT currently integrated):
// leadCreationHelper.ts, leadUpdateHelper.ts, responseBuilder.ts, searchHelperV2.ts
```

## Core Models

### Legacy Models (CommonJS - Being Phased Out)
- `LeadsModel.cjs` - Old lead model
- `contact-model.cjs` - Old contact model

### New Models (TypeScript - Use These!)
- `leadsModelV2.ts` - Normalized lead schema
- `contactModelV2.ts` - Normalized contact schema
- `piiModel.ts` - Personal info (phone, email)
- `addressModel.ts` - Address with deduplication
- `notesModel.ts` - Notes by pii_id
- `queryModel.ts` - Query documents

## Critical Schema Facts

### 1. Phone Field Name
```typescript
// ❌ WRONG
pii.phone_numbers

// ✅ CORRECT
pii.phone_number  // Singular name, but it's an array!
```

### 2. Address vs Lead Fields
```typescript
// Address model uses 'zipcode'
address.zipcode

// Lead model uses 'pincode' (duplicated)
lead.pincode

// Response uses 'pinCode'
response.pinCode
```

### 3. Custom IDs
```typescript
// All models use custom IDs, NOT MongoDB _id
lead.lead_id    // "LEAD-123"
pii.pii_id      // "PII-456"
address.address_id // "ADDR-789"

// Cannot use .populate() - manual joins required
```

### 4. Queries
```typescript
// Stored as array in lead
lead.queries = ["QRY-1", "QRY-2", "QRY-3"]

// Always create new, never update
const query_id = await createQuery(data);
await lead_model.updateOne(
  { lead_id },
  { $push: { queries: query_id } }
);
```

### 5. Notes
```typescript
// Separate collection, referenced by pii_id (NOT lead_id!)
notes_model.find({ pii_id: lead.pii_id })
```

## Common Operations

### Create Lead
```typescript
const lead = await createLead({
  body: req.body,
  agentInfo: {
    agentId: req.agentId,
    agent_category: req.agent_category,
    agent_email: req.agent_email,
    ip_address: req.ip
  }
});
const response = await buildLeadResponse(lead);
```

### Search Leads
```typescript
const results = await searchLeads({
  firstName: "John",
  contact: "9876543210",
  city: "Bangalore",
  page: 1,
  limit: 20
});
const responses = await buildLeadResponseBatch(results.leads);
```

### Update Lead
```typescript
const updated = await updateLead({
  lead_id: "LEAD-123",
  body: req.body,
  agentInfo: {
    agentId: req.agentId,
    agent_category: req.agent_category,
    agent_email: req.agent_email,
    ip_address: req.ip
  }
});
```

## Authentication

Request augmented by `authMiddleware.cjs`:
```typescript
req.agentId           // Agent custom ID
req.agent_category    // Agent department
req.agent_email       // Agent email
req.katyayani_id      // System ID
req.ip                // IP address
```

## Module Interconnections

```
Leads ─→ PII ─→ Address
  │       │
  │       └─→ Notes (by pii_id)
  │
  └─→ Queries (array of query_ids)
  │
  └─→ Contacts (filtered leads)
  │
  └─→ Deals
```

## Project Structure

```
routes/
├── lead/          # Lead endpoints & helpers
├── contact/       # Contact endpoints
├── deals/         # Deal management
├── customer/      # Customer management
└── ...

models/
├── *ModelV2.ts    # New TypeScript models
└── *Model.cjs     # Legacy CommonJS models

utils/
├── piiHelper.ts   # PII utilities
└── ...

middlewares/
├── authMiddleware.cjs
└── ...
```

## Important Documents

- `/MIGRATION_GUIDE.md` - Detailed migration instructions
- `/MIGRATION_STATUS.md` - Current implementation status
- `/QUICK_REFERENCE.md` - Field mappings and examples
- `/claude-brain/.../schema-normalization-migration.md` - Complete overview

## Development

```bash
# Run dev server
npm run dev

# Run tests
npm test

# Build TypeScript
npm run build
```

## Common Gotchas

1. **Phone field**: `phone_number` (singular) but array
2. **Zipcode vs pinCode**: Different names in Address/Lead/Response
3. **Custom IDs**: No .populate(), manual joins only
4. **Queries**: Always create new, append to array
5. **Notes**: Fetch by pii_id, not lead_id
6. **Case transformations**: Names lowercase in storage, State uppercase in response
7. **Deduplication**: Phone → PII, Address1+City → Address

## Quick Field Mapping

| Request | Storage Location | Storage Field |
|---------|------------------|---------------|
| firstName | Lead | first_name (lowercase) |
| contact | PII | phone_number[0] (normalized) |
| email | PII | email (lowercase) |
| address1 | Address | address1 (lowercase) |
| city | Address | city (lowercase) |
| pinCode | Address | zipcode |
| State | Address+Lead | state (lowercase) |

## Response Mapping

| Storage | Response |
|---------|----------|
| first_name | firstName |
| phone_number[0] | contact |
| state | State (uppercase) |
| zipcode | pinCode |
| call_status | callStatus |

---

For detailed information, see individual module documentation in `/claude-brain/projects/CRM-Backend/modules/`
