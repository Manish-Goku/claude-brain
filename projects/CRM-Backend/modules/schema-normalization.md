# Schema Normalization Migration

## Overview
Database schema normalization with TypeScript migration on `schema_normalization` branch. Separates denormalized lead/contact data into normalized PII, Address, Notes, Query collections with custom string IDs.

## Architecture

### Collection Mapping
| Old Collection | New Collection | Model File | Custom ID |
|---|---|---|---|
| `leads` | `lead_v2` | `leadsModelV2.ts` | `lead_id` (K0-XXXX) |
| `contacts` | `contact_v2` | `contactModelV2.ts` | `contact_id` (C0-XXXX) |
| N/A (inline) | `pii_v2` | `piiModel.ts` | `pii_id` (PII-XXXX) |
| N/A (inline) | `address_v2` | `addressModel.ts` | `address_id` (ADDR-XXXX) |
| N/A (inline) | `note` | `notesModel.ts` | (no custom id, uses pii_id ref) |
| N/A (inline) | `query` | `queryModel.ts` | `query_id` (QRY-XXXX) |

### Relationship Map
```
Lead (lead_v2)
├── pii_id → PII (pii_v2)
│   ├── phone_number[] (array, singular name!)
│   ├── email
│   ├── country_code
│   └── addresses[] → Address (address_v2)
│       └── address1 + city = compound unique index
├── queries[] → Query (query_id strings, latest = last)
├── Notes: fetched by pii_id (NOT stored in lead)
├── tags[] (string tag_ids)
└── lead_owner: { agent_id, agent_ref }
```

### Critical Field Name Gotchas
| Context | Field | Notes |
|---|---|---|
| PII model | `phone_number` | SINGULAR name but ARRAY type |
| Address model | `zipcode` | NOT `pinCode` |
| Lead model | `pincode` | Duplicated from Address for perf |
| Response | `pinCode` | Maps from address.zipcode |
| Response | `State` | Capital S, maps from address.state |
| Response | `callStatus` | Maps from lead.call_status |
| Response | `follow_Up_date` | Maps from lead.follow_up_date |
| Lead | `queries` | Array of query_id strings |
| Notes | by pii_id | NOT by lead_id, NOT stored in lead |

### No .populate() — Manual Joins Only
All models use custom string IDs (not MongoDB ObjectId), so `.populate()` cannot be used. All joins are manual via `findOne({ pii_id })`, `find({ address_id: { $in: [...] } })`, etc.

## File Structure
```
routes/lead/
├── route.ts                    # Active route (V2 + legacy combined)
├── route.cjs                   # Old route (superseded by route.ts)
├── controllers/
│   ├── lead_v2.ts              # PRIMARY - V2 schema controller (9 endpoints)
│   ├── lead.ts                 # Scaffold (unused)
│   └── lead.cjs                # OLD controller (legacy endpoints)
├── utils/
│   ├── schemaMapper.ts         # ACTIVE - Maps V2↔old format
│   ├── piiHandler.ts           # ACTIVE - PII create/update/dedup
│   ├── addressHandler.ts       # ACTIVE - Address create/update/dedup
│   ├── queryHandler.ts         # ACTIVE - Query create (always new)
│   ├── notesHandler.ts         # ACTIVE - Notes create/fetch by pii_id
│   ├── searchHelpers.cjs       # ACTIVE - Search query builder (V2-compatible)
│   ├── searchHelperV2.ts       # REFERENCE - Incomplete, missing features
│   ├── responseBuilder.ts      # REFERENCE - Batch response builder (unused)
│   ├── leadCreationHelper.ts   # REFERENCE - Creation orchestrator (unused)
│   ├── leadUpdateHelper.ts     # REFERENCE - Update orchestrator (unused)
│   └── ...other CJS utils
└── validators/
    └── index.cjs               # Express-validator chains
```

## Endpoint Migration Status

### Migrated to V2 (in lead_v2.ts)
| Endpoint | Method | Function | Status |
|---|---|---|---|
| `/search` | GET | search_lead | Complete |
| `/socket-event` | GET | correct_owner | Complete |
| `/all/:leadId` | GET | lead_by_id | Complete |
| `/create` | POST | create_lead | Complete |
| `/call-event` | POST | call_event | Complete |
| `/create/contacts` | POST | create_contacts | Complete |
| `/:leadId` | PUT | update_lead | Complete |
| `/transfer-leads` | PATCH | transfer_leads | Complete |
| `/update-multiple` | PATCH | update_multiple_leads | Complete |

### Still on Legacy Controller (in lead.cjs, routed via route.ts)
| Endpoint | Method | Notes |
|---|---|---|
| `/leadOwner` | GET | Lead owner history |
| `/call-leads` | GET | Call leads list |
| `/search/call-leads` | GET | Search call leads |
| `/create-interact-lead` | POST | InteractBin creation |
| `/statetask` | POST | State task creation |
| `/csvleads` | POST | CSV lead import |
| `/assign/by/csv` | POST | CSV assignment |
| `/assign-call-leads` | POST | Call lead assignment |
| `/assign-retailers` | PATCH | Retailer assignment |
| `/is-enabled` | PATCH | Enable/disable leads |
| `/update-interakt-time/:leadId` | PATCH | Interakt time |
| `/assign` | PATCH | Round-robin assignment |
| `/interakt/:contact` | DELETE | Delete interakt leads |

## Token Optimization Applied
1. **Batch DB operations**: PII fetch for Interakt assignment now uses single `find({ pii_id: { $in: [...] } })` instead of N individual `findOne()` calls
2. **Schema enum extraction**: create_contacts uses `contact_model.schema.path().options.enum` instead of hardcoded arrays
3. **Reusable helpers**: piiHandler, addressHandler, queryHandler, notesHandler used across create/update flows
4. **Promise.all**: PII + agent lookups parallelized in search, socket-event, call-event
5. **Strategic projections**: `.select("pii_id")`, `.select("phone_number")` in search helpers
6. **Bulk writes**: update_multiple_leads uses `bulkWrite()` for batch updates

## Key Decisions
- **searchHelpers.cjs kept** as active search builder — already handles V2 field mapping, has features (customerId lookup, inclusion/exclusion filters, numeric ranges) that searchHelperV2.ts lacks
- **leadCreationHelper/leadUpdateHelper not integrated** — lead_id format mismatch (LEAD-X vs K0-X), controller handles auth/socket/response concerns
- **responseBuilder.ts not integrated** — lead_v2.ts uses schemaMapper.mapLeadToOldFormat + populated_pii_data instead
- **Old .cjs model files deleted** (piiModel.cjs, addressModel.cjs, queryModel.cjs, tagsModel.cjs) — .ts versions are canonical

## Critical Bug Fixes Applied (Feb 2026)
1. **Model import fix**: lead_v2.ts imported `{ leads_model }` from `LeadsModel.cjs` (old model, destructure would be undefined) → fixed to `{ lead_model: leads_model }` from `leadsModelV2`
2. **Contact model import**: Fixed `contactModelV2.cjs` → `contactModelV2` (only .ts exists)
3. **Search helper imports**: Fixed `piiModel.cjs` and `addressModel.cjs` → extensionless paths (`.cjs` files deleted)
4. **Missing endpoints**: route.ts was missing 13 legacy endpoints → added with old controller routing
5. **Console.log cleanup**: Removed debug logs from lead_v2.ts and notesHandler.ts
6. **Enum validation**: Replaced hardcoded enum arrays with schema-derived values
7. **Batch PII fetch**: Interakt assignment loop replaced N findOne calls with single batch find
8. **tagsModel.cjs imports**: Fixed 8 files still referencing deleted `tagsModel.cjs` → extensionless `tagsModel` (resolves to `.ts`)
9. **PiiData type fix**: `customer/utils/upsertHelpers.ts` — `PiiData.phone_number` changed from `string` to `string[]` to match actual pii_interface

## Remaining Work

### TOP PRIORITY — Missing Business Logic in V2 Endpoints
**Ask user which ones to implement before starting work.** These are business logic pieces present in old `lead.cjs` but missing from migrated V2 endpoints in `lead_v2.ts`:

#### update_lead side effects (all missing from V2)
1. **Dispossession handling** (`lead.cjs:333-368`) — calls `dispossession_update()`, creates `DispossessionHistory` records, schedules follow-up cron via `calculateCronBin`, updates `agent_dashboard` (closed/open lead counts for sales agents)
2. **Query update handling** (`lead.cjs:327-331` + `QueryUpdate.cjs`) — validates query category against Tags, pushes to lead's `query` array, emits socket `updated-lead`
3. **Call status side effects** (`lead.cjs:434-459, 505-509`) — sets `callTime` (IST), calculates `nextPriorityUpdate`, schedules cron bin via `calculateCronBin`, resets `priority` (15 or 10), sets `leadStatus` to "Existing Lead", pushes to `callStatusHistory`
4. **`user_message_time` sync to Contacts** (`lead.cjs:371-400`) — compares dates, updates Contact doc's `user_message_time`, pushes audit trail to `updated_data`
5. **`leadProfile` → Customer profile update** (`lead.cjs:427-430`) — calls `update_customer_profile()` to sync Customer model's `customer_profile` field
6. **`order_history` merge** (`lead.cjs:406-412`) — appends new entries to existing array (not replace)
7. **Tags processing on update** (`lead.cjs:479-503`) — inserts new tags into Tags collection, uses `$addToSet` on lead
8. **`is_enabled` protection** (`lead.cjs:402`) — `delete req.body.is_enabled` prevents external toggle

#### create_lead missing logic
9. **RLMS/Retailer dual check** (`lead.cjs:162-242`) — parallel check against CRM Leads + RLMS Retailer collection, 4-case decision tree (both exist/CRM only/RLMS only/neither), disables CRM lead if RLMS is older
10. **Initial priority scoring** (`lead.cjs:248-249`) — `calculateInitialPriority(leadStatus, leadSubType)` from `leadScoring.cjs`

#### transfer_leads missing logic
11. **Cross-entity transfer** (`lead.cjs:1294-1296`) — old calls `transfer_everything()` which updates ownership across Leads + Contacts + Deals + Quotations. V2 only updates `lead_v2`.

### Other Remaining Work
- [ ] Migrate legacy endpoints from lead.cjs to lead_v2.ts (13 endpoints)
- [ ] Complete searchHelperV2.ts with missing features (customerId, inclusion filters, numeric ranges)
- [ ] Decide if leadCreationHelper/leadUpdateHelper should be the canonical path
- [x] ~~Pre-existing TS error in customer/utils/upsertHelpers.ts~~ — Fixed: `PiiData.phone_number` changed from `string` to `string[]`
