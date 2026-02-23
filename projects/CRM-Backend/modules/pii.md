# PII Module

## Purpose
Centralizes personal identifiable information (phone numbers, addresses, GST numbers) that was previously duplicated across leads, deals, contacts, customers, and quotations. Other modules reference a single PII record instead of storing contact details inline.

## First TypeScript Route Module
This is the first route module written in TypeScript (.ts). Uses `require()` for CJS imports, `module.exports` for route export, Express types from `@types/express`.

## File Structure
```
routes/pii/
├── route.ts                         — Route definitions (auto-mounted at /pii)
├── controller/
│   └── pii.ts                       — All controllers (PII + Address + GST)
└── validators/
    ├── index.ts                     — Barrel re-exports
    ├── piiValidators.ts             — PII and GST validation chains
    └── addressValidators.ts         — Address validation chains

models/
├── pii_model.cjs                    — PII schema (phone, country_code, addresses, gst_numbers)
└── address_model.cjs                — Address schema (address lines, city, state, zipcode)
```

## Models

### PII (`pii`)
- `pii_id`: auto-generated `PII001`, `PII002` via sequence service
- `phone_number`: unique, 6-15 digits (no country code)
- `country_code`: default `+91`, format `+X` to `+XXXX`
- `addresses`: array of ObjectId refs to `Address` model
- `gst_numbers`: embedded array — `{ gst_number, business_name, status, registration_date }`
- `extension`: optional string
- `created_at`, `updated_at`: timestamps

### Address (`Address`)
- `address_id`: auto-generated `ADDR0001`, `ADDR0002` via sequence service
- `address1` (required), `address2`, `address3`
- `city`, `state`, `zipcode` (required), `country` (default: India)
- `is_primary`: boolean (only one per PII)
- `channels`: array of strings

## API Endpoints

### PII CRUD
| Method | Path | Description |
|--------|------|-------------|
| GET | `/pii/` | List PIIs (paginated, filter by phone_number/pii_id) |
| GET | `/pii/:piiId` | Get single PII with populated addresses |
| POST | `/pii/` | Create PII |
| PATCH | `/pii/:piiId` | Update PII fields |
| DELETE | `/pii/:piiId` | Delete PII + all associated addresses |

### Address (nested under PII)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/pii/:piiId/address` | Add address to PII |
| PATCH | `/pii/:piiId/address/:addressId` | Update address |
| DELETE | `/pii/:piiId/address/:addressId` | Remove address |

### GST (nested under PII)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/pii/:piiId/gst` | Add GST number |
| PATCH | `/pii/:piiId/gst/:gstNumber` | Update GST entry |
| DELETE | `/pii/:piiId/gst/:gstNumber` | Remove GST entry |

## Key Business Rules
- Phone number must be globally unique across all PIIs
- GST numbers are checked for global uniqueness (across all PII records)
- Setting `is_primary: true` on an address automatically unsets `is_primary` on all other addresses of that PII
- Deleting a PII also deletes all referenced Address documents
- Address ownership is verified (address must belong to the PII) before update/delete

## Bug Fix Applied (Feb 2026)
- Fixed model name mismatch: `address_model.cjs` registered as `'address'` but `pii_model.cjs` used `ref: 'Address'` — changed registration to `'Address'` to match
