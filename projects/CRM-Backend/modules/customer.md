# Customer Module

## Overview
Core CRM module for managing customer records — creation, search, assignment, status updates, franchise grouping, and prescription uploads.

## TypeScript Migration (Feb 2026)
Migrated from `.cjs` to `.ts`. Both versions coexist:
- `routes/customer/route.ts` — preferred by route loader
- `routes/customer/controllers/customer.ts` — ~1700 lines, 18 exports
- Original `.cjs` files kept as fallback

## File Structure
```
routes/customer/
├── route.ts              # TS route (preferred)
├── route.cjs             # CJS fallback
├── controllers/
│   ├── customer.ts       # TS controller (preferred)
│   └── customer.cjs      # CJS fallback
├── validators/
│   └── index.cjs         # express-validator chains
├── sanitizers/
│   └── index.cjs         # Input sanitizers
└── service/
    └── customer_service.cjs  # Business logic helpers
```

## API Endpoints
| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| GET | `/all` | allCustomer | List all customers (paginated) |
| GET | `/all/:customerId` | allCustomer | Get single customer |
| GET | `/search` | searchCustomer | Search customers |
| GET | `/sort` | sortedCustomers | Sort customers |
| GET | `/franchise/preSignedUrl` | getPreSignedUrl | S3 pre-signed URL |
| GET | `/franchise/:franchiseId` | getCustomerforFranchise | Customers by franchise |
| POST | `/create` | createCustomer | Create customer |
| POST | `/contacts` | createContacts | Create contacts (restricted) |
| POST | `/uploadPrescription` | uploadPrescription | Upload prescription files |
| PUT | `/update-multiple` | updateMultiple | Bulk update customers |
| PUT | `/leadstatus` | updateStatusLead | Update lead status |
| PUT | `/:customerId` | updateCustomer | Update single customer |
| PATCH | `/assign` | assign_customer | Assign customer to agent |
| POST | `/cancel-order` | cancelled_customer_order | Cancel/RTO customer order (changed from PATCH → POST, Feb 2026) |
| DELETE | `/:customerId` | deleteCustomer | Delete customer |

## Key Dependencies
- Models: `customer_model`, `agent_model`, `franchise_model`, `leads_model`, `deals_model`, `contacts_model`, `customer_order`
- Services: `customer_service.cjs`, `franchise_service.cjs`, `lead_service.cjs`, `event_service.cjs`
- Utils: S3 (`@utils/s3.cjs`), error handling, audit trail

## Middleware
- Validators from `./validators/index.cjs`
- Sanitizers from `./sanitizers/index.cjs`
- `logRequest`, `verifyToken` on all routes
- `restrictTo(["Super Admin", "Admin", "Floor Manager"])` on `/contacts`
- `multer` memory storage for `/uploadPrescription` (max 10 files)
