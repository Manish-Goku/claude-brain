# Token Service — Centralized Token Management

## Overview
Centralized API for backend services to obtain authentication tokens for third-party platforms. Two endpoint categories: **dynamic** tokens (refreshable, cached with thread-safe managers) and **static** tokens (env-var secrets returned directly). All backed by thread-safe token managers with deduplication.

---

## API Routes

### Dynamic Tokens — `GET /api/tokens/dynamic`
**Auth:** JWKS JWT middleware
**Query params:** `platform` (required), `force_refresh` (optional, "true"/"false")

| Platform | Auth Type | Token Source | Caching |
|----------|-----------|-------------|---------|
| `shiprocket` | Bearer | Shiprocket login API | `RefreshableTokenManager`, 1-min buffer |
| `delhivery-b2b` | Bearer | Delhivery B2B JWT login | `DelhiveryB2BTokenManager`, 30s buffer, 2h expiry check |

### Static Tokens — `GET /api/tokens/static`
**Auth:** JWKS JWT middleware + IP whitelist (`ALLOWED_IPS` env var)

| Platform | Auth Type | Token Source |
|----------|-----------|-------------|
| `delhivery-b2c` | Bearer | `DELHIVERY_AUTH_TOKEN_B2C` env var |
| `razorpay` | Basic | Base64(`RAZORPAY_KEY_ID:RAZORPAY_KEY_SECRET`) |

**Route file:** `routes/tokenRoutes.go`

---

## Response Format

```json
{
  "platform": "shiprocket",
  "auth_type": "bearer",
  "token": "eyJ...",
  "expires_at": "2026-02-24T15:00:00Z",
  "cached": true
}
```
- `expires_at`: ISO 8601, null for static tokens and Shiprocket (no expiry exposed)
- `cached`: true if token returned from cache, false if freshly obtained
- `auth_type`: "bearer" or "basic" — tells client which Authorization scheme to use

---

## Token Managers (`utils/token_manager.go`)

### RefreshableTokenManager (generic, used by Shiprocket)
```go
type RefreshableTokenManager struct {
    token, lastRefreshTime, refreshing, refreshCond, refreshFunc
}
```
- Constructor takes a `refreshFunc(client) → (token, error)`
- `RefreshToken(client, minRefreshInterval, forceRefresh)`:
  - If refreshed within `minRefreshInterval` → return cached (even if forceRefresh)
  - If another goroutine refreshing → wait via `sync.Cond`
  - Otherwise → call `refreshFunc` outside lock → cache result → broadcast

### DelhiveryB2BTokenManager (specialized for JWT tokens)
Same dedup pattern as above, plus:
- `RefreshTokenIfNeeded(client, minRefreshInterval, expiryBuffer)`:
  - Check 1: refreshed within minRefreshInterval → cached
  - Check 2: JWT `exp` claim still valid (> now + expiryBuffer) → cached
  - Check 3: another goroutine refreshing → wait
  - Otherwise → refresh
- `checkJWTExpiry(token, buffer)` — parses JWT `exp` claim unverified
- `GetExpiryFromJWT(token)` — returns RFC3339 expiry string

### TokenCache (simple read/write)
Thread-safe string cache with Get/Set/Clear. Used as legacy `ShiprocketTokenCache`.

---

## Platform-Specific Token Logic (`utils/token_helper.go`)

### Shiprocket
- **Login:** POST `https://apiv2.shiprocket.in/v1/external/auth/login` with email/password
- **Response:** `{"token": "..."}`
- **Validation:** GET `/v1/external/orders` with Bearer token — if 401, token invalid
- **Env vars:** `SHIPROCKET_EMAIL`, `SHIPROCKET_PASSWORD`
- **Two paths:** `GetShiprocketToken()` (validate-then-refresh, used by tracking) and `ShiprocketTokenManager.RefreshToken()` (used by token API)

### Delhivery B2B
- **Login:** POST `https://ltl-clients-api.delhivery.com/ums/login` with username/password
- **Response:** `{"data": {"jwt": "..."}}`
- **JWT expiry:** Parsed from `exp` claim (unverified), 2-hour buffer
- **Env vars:** `DELHIVERY_USERNAME`, `DELHIVERY_PASSWORD`
- **Two paths:** `DelhiveryToken(client)` (legacy, used by tracking) and `B2BTokenManager.RefreshTokenIfNeeded()` (used by token API)

### Delhivery B2C
- **Static:** Returns `DELHIVERY_AUTH_TOKEN_B2C` env var directly
- No refresh logic

### Razorpay
- **Static:** Encodes `RAZORPAY_KEY_ID:RAZORPAY_KEY_SECRET` as Base64
- Auth type: "basic" (client uses `Authorization: Basic {token}`)
- **Env vars:** `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`

---

## IP Whitelist Middleware

**Function:** `IPWhitelistMiddleware()` in `handlers/token_handler.go`
- Applied to static token endpoint only
- Reads `ALLOWED_IPS` env var (comma-separated)
- `0.0.0.0` = allow all
- Empty = allow all (with log warning)
- Non-matching IP → 403 Forbidden

---

## Key Files

| File | Purpose | Lines |
|------|---------|-------|
| handlers/token_handler.go | Dynamic/static handlers, IP whitelist, response types | 307 |
| utils/token_manager.go | RefreshableTokenManager, DelhiveryB2BTokenManager, TokenCache | 320 |
| utils/token_helper.go | Shiprocket login, Delhivery B2B login, token validation | 201 |
| utils/sso_token_manager.go | SSOTokenManager (used by cron jobs, not token API) | 298 |
| routes/tokenRoutes.go | Route registration | 28 |

---

## Env Vars

| Var | Used By |
|-----|---------|
| `SHIPROCKET_EMAIL`, `SHIPROCKET_PASSWORD` | Shiprocket token |
| `DELHIVERY_USERNAME`, `DELHIVERY_PASSWORD` | Delhivery B2B token |
| `DELHIVERY_AUTH_TOKEN_B2C` | Delhivery B2C static token |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | Razorpay basic auth |
| `ALLOWED_IPS` | IP whitelist for static endpoint |
