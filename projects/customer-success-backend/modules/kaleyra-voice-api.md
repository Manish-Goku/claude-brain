# Kaleyra Voice API — Click-to-Call + Outbound Calling

**Date:** 2026-02-23 (created), 2026-02-24 (rewritten for legacy API)
**Status:** Rewritten for legacy api-voice.kaleyra.com, ready for testing

## Commit History

| Commit | Description |
|--------|-------------|
| `80383c4` | Original kaleyra.io implementation (SID-based, `/v1/{SID}/voice/` URLs) — **PRESERVE as fallback if account migrates to kaleyra.io** |
| `210f7d4` | Rewritten for legacy api-voice.kaleyra.com (`POST /v1/` with `method` param, no SID) |

## Architecture (Legacy API)

```
Agent clicks "Call" on frontend
  → POST /calls/click-to-call (our backend)
    → POST https://api-voice.kaleyra.com/v1/ (method=dial.click2call)
      → Kaleyra dials agent (caller) → bridges to customer (receiver)
      → Callback: GET /webhooks/kaleyra/callback?caller=X&receiver=Y&status=ANSWER&...
        → Update ivr_calls → Supabase realtime → Frontend updates
```

## Key Differences from kaleyra.io (commit 80383c4)

| Aspect | Old (kaleyra.io) | New (legacy) |
|--------|------------------|--------------|
| Base URL | `api.kaleyra.io/v1/{SID}/voice/click-to-call` | `api-voice.kaleyra.com/v1/` (single URL) |
| Routing | Different URL paths per action | `method` form param (`dial.click2call`, `voice.call`, etc.) |
| Auth | `api-key` header + SID | `api_key` form param (no SID) |
| Params | `from`, `to`, `bridge` | `caller`, `receiver` (no bridge) |
| Callback | POST with JSON body | GET with URL template placeholders |
| Statuses | `from_call_start`, `to_call_answer`, `call_end` | `ANSWER`, `BUSY`, `NOANSWER`, `CANCEL`, `FAILED`, `CONGESTION` |

## Files

| File | Purpose |
|------|---------|
| `src/kaleyraVoice/kaleyraVoice.service.ts` | API client — click_to_call, outbound_call, process_callback, get_call_logs |
| `src/kaleyraVoice/kaleyraVoice.controller.ts` | 4 endpoints |
| `src/kaleyraVoice/kaleyraVoice.module.ts` | Module registration |
| `src/kaleyraVoice/dto/clickToCall.dto.ts` | DTO: customer_number, agent_number, agent_name |
| `src/kaleyraVoice/dto/outboundCall.dto.ts` | DTO: customer_number, play (sound file), campaign |
| `src/kaleyraVoice/dto/callLogs.dto.ts` | DTO: from_date, to_date, call_to, page, limit |

## Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/calls/click-to-call` | Initiate click-to-call (caller=agent, receiver=customer) |
| POST | `/calls/outbound` | OBD call with sound file/IVR |
| GET | `/webhooks/kaleyra/callback` | Kaleyra callback (GET with query params) |
| GET | `/calls/logs` | Fetch C2C call history from Kaleyra |

## Callback Status Mapping

| Kaleyra Status | ivr_calls status |
|----------------|-----------------|
| `ANSWER` | `answered` |
| `BUSY` | `busy` |
| `NOANSWER` | `missed` |
| `CANCEL` | `cancelled` |
| `FAILED` | `failed` |
| `CONGESTION` | `failed` |

## Callback URL Template

Kaleyra replaces `{placeholder}` vars before hitting our endpoint:
```
GET /webhooks/kaleyra/callback?caller={caller}&receiver={receiver}&status={status}&status1={status1}&status2={status2}&duration={duration}&billsec={billsec}&starttime={starttime}&endtime={endtime}&recordpath={recordpath}&callerid={callerid}&id={id}
```

## .env (only 2 vars needed)

```
KALEYRA_API_KEY=2148ee941108435205a69d8eec78eafb
KALEYRA_CALLBACK_URL=http://localhost:3002/webhooks/kaleyra/callback
```

## Testing

```bash
# Click-to-call
curl -X POST http://localhost:3002/calls/click-to-call \
  -H 'Content-Type: application/json' \
  -d '{"customer_number":"9876543210","agent_number":"8602733437","agent_name":"Manish"}'

# Callback needs public URL — use ngrok:
# ngrok http 3002 → set KALEYRA_CALLBACK_URL=https://<id>.ngrok.io/webhooks/kaleyra/callback
```

## Pending

1. **Public callback URL** — ngrok or deployed URL (localhost won't receive callbacks)
2. **End-to-end test** — click-to-call → verify agent phone rings → call completes → check ivr_calls
3. **Frontend wiring** — documented in `~/Desktop/frontend-tasks.md`

## Important: Kaleyra does NOT support browser calling

Agent still talks on mobile phone. Laptop only initiates the call.
For true browser-based calling → Exotel/Twilio WebRTC (see `~/Desktop/browser-calling-research.md`).
