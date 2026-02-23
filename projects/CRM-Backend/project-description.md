# CRM-Backend

## Overview
Backend service for Katyayani Organics' CRM system.

## Tech Stack
- **Runtime:** Node.js, Express.js
- **Language:** JavaScript (CommonJS .cjs) — gradual TypeScript migration in progress
- **Database:** MongoDB (Mongoose ODM)
- **TS Runtime:** `tsx` (esbuild-powered) — chosen over `node --experimental-strip-types` for extensionless CJS require resolution

## TypeScript Setup (Feb 2026)
Gradual migration — `.ts` and `.cjs` coexist. TS files can import CJS, CJS files can `require()` TS files (via tsx).

### Key Config
- `tsconfig.json` — `module: "CommonJS"`, `allowJs: true`, `strict: false`, path aliases (`@models/*`, `@middlewares/*`, `@utils/*`, `@config/*`)
- `types/express.d.ts` — augments Express `Request` with auth middleware properties (`agentId`, `katyayani_id`, `agent_email`, `agent_slack_id`, `agent_role_name`, `superAdmin`, `token`, `user`, `clientIP`)
- `types/module-alias.d.ts` — declaration for `module-alias/register`
- `jest.config.js` — `ts-jest` transform for `.test.ts` files, `moduleFileExtensions` includes `ts`

### Scripts
| Script | Command | Purpose |
|--------|---------|---------|
| `dev` | `tsx watch app.js` | Dev with hot reload + TS support |
| `start:tsx` | `tsx app.js` | Run via tsx (staging/prod) |
| `start` | `node app.js` | Original Node.js start (unchanged) |
| `typecheck` | `tsc --noEmit` | Type checking for CI |
| `build` | `tsc` | Compile to dist/ |

### Route Loader (`routes/index.cjs`)
`loadRoutes` discovers both `route.cjs` and `route.ts`. If both exist in the same directory, prefers `route.ts` (migration scenario).

### How to Migrate a Route to TypeScript
1. Create `route.ts` alongside `route.cjs` in the route directory
2. The route loader auto-picks `route.ts` over `route.cjs`
3. Once verified, delete the old `route.cjs`

## Status
- Active project
- TypeScript infrastructure set up, gradual migration in progress

## Migrated Modules (TypeScript)
- [PII](./modules/pii.md) — Centralized PII (phone, address, GST) — first TypeScript route module (created in TS)
- [Customer](./modules/customer.md) — Core customer CRUD, search, assignment — migrated from CJS, both versions coexist
- [Schema Normalization](./modules/schema-normalization.md) — Database schema normalization (Lead/Contact → PII/Address/Notes/Query). Branch: `schema_normalization`. 9 core endpoints migrated to V2, 13 legacy endpoints still on old controller.

## Other Modules
- [Agronomy Test](./modules/agronomy-test.md) — Test management for sales agents (CJS)
- [Leads](./modules/leads.md) — Lead management API contract (20+ endpoints)
- [Contacts](./modules/contacts.md) — Contact management API contract
- [Deals](./modules/deals.md) — Deal pipeline management
- [Quotations](./modules/quotations.md) — Quotation management
