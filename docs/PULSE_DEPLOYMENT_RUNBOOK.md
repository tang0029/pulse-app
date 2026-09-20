# Pulse — Deployment Runbook

First deploy to production, and what to check afterwards. Follow in order.

Local setup is in the repo [`README.md`](../README.md). Architecture is in [`PULSE_HLD.md`](./PULSE_HLD.md).

> **Why this is a separate document:** step 3 (SPF/DKIM/DMARC) and step 9 (the production smoke test) are the difference between alerts arriving and alerts landing silently in a spam folder. The product's entire safety case rests on them. A checklist buried in a spec does not get followed.

## First deploy

1. Create Supabase project in **ap-southeast-2**.
2. `supabase link` + `supabase db push` — the same migrations that ran locally.
3. Configure Resend: verified sending domain, **SPF + DKIM + DMARC records**. Without these, alert emails land in spam and the safety case fails silently.
4. Set Edge Function secrets (Resend key, webhook signing secret, service role key).
5. Enable `pg_cron`; schedule `dispatch-alerts` at `* * * * *`.
6. Register the Resend webhook against the deployed `email-webhook` URL.
7. Deploy frontend to Netlify; env vars = project URL + anon key **only**.
8. Disable public signup; invite the four accounts manually.
9. **Smoke test in production:** log a deliberately high reading from a real parent account, confirm the email arrives in the UK family member's actual inbox — check the spam folder — then delete the test reading.
10. Only then, hand the URL to your parents.

## After deploy — what to check, and when

| When | Check | Where |
|---|---|---|
| Day 1 | Both parents signed in successfully, sessions persisted | Supabase Auth dashboard |
| Day 1 | A real reading saved by each parent | `/health` route |
| Week 1 | Any `reading_save_failed` or `alert_dispatch_failed` events | `/health` route — non-zero is a bug, investigate |
| Week 1 | Logging consistency per parent | `/health` route |
| Monthly | Manual CSV export taken and stored | Supabase SQL editor |
| Once, before relying on it | A backup actually restores | Supabase PITR |

## When an alert doesn't arrive

Work down the chain in this order — each step tells you where it stopped:

1. Did the reading breach? Check `systolic > 150` on the row.
2. Did the rule fire? `select * from app_events where event_name = 'alert_evaluated'` — look for `fired: true`.
3. Was a delivery queued? `select * from alert_deliveries where reading_id = '...'`.
4. Did the dispatcher run? Check `pg_cron` job history and the `dispatch-alerts` function logs.
5. Did Resend accept it? Check `provider_message_id` and `last_error` on the delivery row.
6. Did it bounce or get filtered? Check `status` (fed by the webhook) and the Resend dashboard.
7. Still nothing? Check the spam folder, then re-verify DKIM/DMARC alignment.

## Rolling back

- **Frontend:** redeploy the previous Netlify build — instant, no data impact.
- **Database migration:** Postgres migrations here are additive. Do **not** roll a migration back against production with live readings in it; write a forward migration instead.
- **Alerting:** to stop alerts without a deploy, set `alert_rules.active = false`. Remember to set it back — readings will keep saving, but nobody will be told about a high one.
