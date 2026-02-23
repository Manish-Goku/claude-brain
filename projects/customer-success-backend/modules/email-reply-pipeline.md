# Email Reply Pipeline (2026-02-23)

## What was built
Full outbound email reply: SMTP send via Gmail → persist in DB → thread grouping in frontend.

## Key decisions
- **Same `emails` table** — outbound replies stored alongside inbound, distinguished by `direction` column
- **SMTP reuses IMAP credentials** — same Gmail app password (decrypted from `support_emails.imap_password`)
- **Thread grouping in frontend** — `groupRowsIntoEmails()` groups rows by `thread_id || message_id`
- **Realtime = refetch+regroup** — on INSERT/UPDATE, hook refetches full page instead of manual merge
- **CC/BCC support** — optional arrays in DTO, passed to nodemailer + persisted

## New files
- `src/gmailIngestion/smtp.service.ts` — nodemailer wrapper
- `src/gmailIngestion/dto/sendReply.dto.ts` — validation DTO

## Modified files
- `gmailIngestion.service.ts` — added `send_reply()` method
- `gmailIngestion.controller.ts` — added `POST /emails/:id/reply`
- `gmailIngestion.module.ts` — registered SmtpService
- `gmailTypes.ts` + `emailResponse.dto.ts` — added direction, agent_name, in_reply_to
- Frontend: `useEmails.ts` (groupRowsIntoEmails, send_reply, realtime refetch)
- Frontend: `Emails.tsx` (handleReply wired, CC/BCC UI, loading spinner, selectedEmail sync)
- Frontend: `types.ts` (Supabase typed client updated)

## DB migration
```sql
ALTER TABLE emails ADD COLUMN direction text NOT NULL DEFAULT 'inbound';
ALTER TABLE emails ADD COLUMN agent_name text;
ALTER TABLE emails ADD COLUMN in_reply_to text;
CREATE INDEX idx_emails_in_reply_to ON emails (in_reply_to) WHERE in_reply_to IS NOT NULL;
```
