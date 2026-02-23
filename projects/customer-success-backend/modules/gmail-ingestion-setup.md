# Gmail Ingestion — Setup Session (2026-02-22)

## Completed
- Scaffolded NestJS project with Supabase integration
- Created all source files: gmail.service, gmailIngestion.service/controller, gmailWebhook.controller, gmailCron.service, emailGateway
- All DTOs with Swagger decorators
- Supabase tables created: `support_emails`, `emails`
- Swagger UI at `/api/docs` — all 11 routes verified
- Type check passes (0 errors)
- Server starts successfully on port 3002

## GCP Setup — BLOCKED (needs permissions)
Manish needs to get permission to assign **Pub/Sub Editor** role in GCP project `ultra-glyph-488212-v1`.

### Remaining GCP steps:
1. Create service account `email-ingestion` → assign Pub/Sub Editor role
2. Download JSON key → extract `client_email` + `private_key`
3. Enable domain-wide delegation on service account
4. In Google Workspace Admin: authorize client ID with scope `gmail.readonly`
5. Create Pub/Sub topic `gmail-push-notifications` → grant Publisher to `gmail-api-push@system.gserviceaccount.com`
6. Create push subscription → endpoint: `https://<domain>/webhooks/gmail`
7. For local dev: `ngrok http 3002`
8. Fill `.env` vars: `GOOGLE_SERVICE_ACCOUNT_EMAIL`, `GOOGLE_PRIVATE_KEY`, `GOOGLE_CLOUD_PROJECT_ID`, `GMAIL_PUBSUB_TOPIC`, `APP_PUBLIC_URL`

### Testing checklist:
- [ ] POST `/support-emails` with a workspace email → watch starts
- [ ] Send test email to that address
- [ ] Verify Pub/Sub push hits `/webhooks/gmail`
- [ ] Verify email stored in Supabase `emails` table
- [ ] Verify WebSocket `new_email` event fires
- [ ] Verify cron renews watches (check logs after 6 hours or trigger manually)
