# Cron Jobs Module — Dynamic HTTP Scheduler

## Overview
REST API for managing scheduled HTTP jobs. Jobs are persisted in PostgreSQL (GORM) and executed by an in-memory cron scheduler (`robfig/cron`). On startup, all jobs are restored from DB and re-registered. Each job fires an HTTP request to a configured URL on a cron schedule, authenticated with an SSO Bearer token.

**Database:** PostgreSQL table `job_models` (GORM, but NOT in AutoMigrate — table created separately)
**Scheduler:** `robfig/cron/v3` (in-memory, restored from DB on boot)

---

## Data Model

### JobModel (PostgreSQL via GORM)
```
ID (int32, PK), URL (string, NOT NULL), Method (string, NOT NULL, default: GET),
RepeatTime (string, NOT NULL), StartTime (string, NOT NULL),
CronSpec (string, NOT NULL, indexed), CreatedAt, UpdatedAt
```

### Job (in-memory runtime)
```
ID (int32), URL, Method, RepeatTime, StartTime, CronSpec
```

### Global State (`utils/cron_helper.go`)
```go
JobsMap        map[int32]*Job           // in-memory job registry
CronEntriesMap map[string]cron.EntryID  // cronSpec → scheduler entry ID
Mu             sync.Mutex               // protects JobsMap
CronScheduler  *cron.Cron               // robfig scheduler instance
```

---

## API Routes (`/cron/jobs`, NO auth middleware)

| Method | Path | Handler | Notes |
|--------|------|---------|-------|
| GET | `/` | GetJobsHandler | List all jobs from DB |
| POST | `/` | AddJobHandler | Create + schedule job |
| PUT | `/:id` | UpdateJobHandler | Update + reschedule if cron changed |
| DELETE | `/:id` | DeleteJobHandler | Delete + unschedule if no other jobs share spec |

**Route file:** `routes/cronRoutes.go`
**Registered in:** `routes/route.go` (no JWKS middleware — public endpoints)

---

## Cron Spec Generation (`utils.GenerateCronSpec`)

Converts human-friendly `startTime` + `repeatTime` to cron expression:

| repeatTime | startTime | Generated Cron Spec |
|-----------|-----------|-------------------|
| < 60 min (e.g., "30m") | any | `*/30 * * * *` (every 30 min) |
| 1-23 hours (e.g., "2h") | "14:30" | `30 */2 * * *` (at :30 every 2 hrs) |
| >= 24 hours (e.g., "24h") | "09:00" | `0 9 * * *` (daily at 09:00) |

**startTime format:** `HH:MM` (24h) — used for minute/hour in cron spec
**repeatTime format:** Go duration string (`30m`, `1h`, `24h`, etc.)

---

## Job Execution Flow

When a cron fires:
1. `GetJobsByCronSpec(spec)` — finds all jobs sharing that cron spec
2. `SSOToken.GetToken(1h)` — gets cached SSO Bearer token (refreshes if expiring within 1 hour)
3. For each job: fires HTTP request (`job.Method` to `job.URL`) with `Authorization: Bearer {token}`
4. HTTP client timeout: 60 seconds
5. Logs success/failure per job

**Key design:** Multiple jobs can share the same cron spec — only one cron entry is registered, executing all matching jobs.

---

## SSO Token Manager (`utils/sso_token_manager.go`)

Thread-safe singleton `SSOToken` for authenticating cron HTTP requests.

### Two-Step Login Flow
1. **POST** `{SSO_BASE_URL}/login` with email/password → receives `code`
2. **POST** `{SSO_BASE_URL}/generate-tokens` with code + platform "utility-golang" → receives `access_token` + `refresh_token`

### Token Caching
- Cached in memory with parsed JWT expiry
- Refreshes when token expires within buffer (default 1 hour)
- Thread-safe: if multiple goroutines need token simultaneously, only one refreshes (others wait via `sync.Cond`)
- Retry: 3 attempts with exponential backoff (2s, 4s, 6s)

### Env Vars
- `SSO_BASE_URL` — SSO service URL
- `SSO_EMAIL` — Service account email
- `SSO_PASSWORD` — Service account password

### Methods
- `GetToken(bufferDuration)` — returns cached or refreshed token
- `ForceRefresh()` — clears cache, forces new login
- `ClearToken()` — clears cache
- `GetTokenInfo()` — debug info (expiry, validity, last refresh)

---

## Startup Sequence (`main.go`)

```
InitCron()      → creates robfig/cron scheduler, starts it
RestoreJobs()   → loads all JobModel from DB, populates JobsMap, registers cron entries
```

---

## Job Lifecycle

### Add
1. Validate payload (URL, Method, RepeatTime, StartTime required)
2. Generate cron spec
3. Check for duplicate (same URL + same cron spec → 409 Conflict)
4. Insert into PostgreSQL
5. Add to `JobsMap`
6. Register cron entry (or reuse existing if spec already registered)

### Update
1. Find job by ID in DB
2. Generate new cron spec
3. Update DB record
4. Update `JobsMap`
5. If cron spec changed:
   - Check if old spec still has other jobs → if not, remove old cron entry
   - Register new cron entry

### Delete
1. Find job by ID in DB
2. Delete from DB
3. Remove from `JobsMap`
4. Check if deleted job's cron spec still has other jobs → if not, remove cron entry

---

## Key Files

| File | Purpose | Lines |
|------|---------|-------|
| models/cron_model.go | Job, JobPayload, JobModel structs | 37 |
| handlers/cron_handler.go | CRUD handlers (Add, Update, Delete, Get) | 201 |
| utils/cron_helper.go | InitCron, GenerateCronSpec, AddJobToCron, RestoreJobs, global state | 139 |
| utils/sso_token_manager.go | SSOTokenManager (two-step login, caching, thread-safe) | 298 |
| routes/cronRoutes.go | Route registration | 17 |

---

## Notes
- **No auth on API endpoints** — `/cron/jobs` routes have no JWKS middleware (unlike webhooks/pincode-cases)
- **JobModel not in AutoMigrate** — table `job_models` must exist in PostgreSQL already
- **Cron spec deduplication** — multiple jobs can share one cron entry; entry removed only when last job with that spec is deleted
- **SSO token is per-batch** — fetched once per cron tick, shared across all jobs in that tick
