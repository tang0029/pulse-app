# Pulse — High-Level Design

Architecture for **Pulse**, a blood pressure tracking web app for one family: two patients in New Zealand, two family viewers (one in the UK, one in NZ).

- **What and why:** [`PULSE_PRD.md`](./PULSE_PRD.md)
- **Schema, policies, triggers, functions:** [`PULSE_LLD.md`](./PULSE_LLD.md)
- **Scope, analytics, security, tests, acceptance criteria:** `SPEC.md`
- **Deploying it:** [`PULSE_DEPLOYMENT_RUNBOOK.md`](./PULSE_DEPLOYMENT_RUNBOOK.md)

## The requirement that shapes the architecture

An out-of-range reading logged in New Zealand must reach a family member in the UK **the same day, without them opening the app**.

That single requirement forces everything below: a server (not a browser-only app), a persistent store, a rule evaluated on every write, a transactional email provider, and delivery tracking so a failed email is visible rather than silent. It also rules out end-to-end encryption — a server that cannot read a systolic value cannot decide whether to send an alert.

Two invariants the architecture must hold:

1. A reading is never lost — it is written durably, or the user is told it wasn't.
2. An alert is never silently dropped — if an email fails, that failure is visible.

---

## 1. Context

```
   NZ                                                UK
┌──────────────┐  ┌──────────────┐          ┌──────────────┐
│  Dad         │  │  Mum         │          │  Me       │
│  (patient)   │  │  (patient)   │          │  (family)    │
└──────┬───────┘  └──────┬───────┘          └──────┬───────┘
       │ logs            │ logs                    │ views + receives alerts
       │                 │                         │
       └────────┬────────┘          ┌──────────────┘   ┌──────────────┐
                │                   │                  │  Sister (NZ) │
                ▼                   ▼                  └──────┬───────┘
        ┌───────────────────────────────────────┐             │
        │        Pulse web app (React SPA)      │◄────────────┘
        │        static hosting / CDN           │
        └───────────────────┬───────────────────┘
                            │ HTTPS, JWT-authenticated
                            ▼
        ┌───────────────────────────────────────┐
        │  Supabase project (ap-southeast-2)    │
        │  ├─ Auth (magic link, allow-listed)   │
        │  ├─ Postgres + Row Level Security     │◄── RLS *is* the API boundary
        │  ├─ Trigger: evaluate_alert_rule()    │
        │  ├─ pg_cron → dispatch-alerts (1 min) │
        │  └─ Edge Functions                    │
        └───────────┬───────────────────────────┘
                    │ REST
                    ▼
        ┌───────────────────────────────────────┐
        │  Transactional email provider         │
        │  (Resend) — send + delivery webhook   │
        └───────────────────────────────────────┘
```

## 2. Stack

Following the PRD's own "e.g." suggestions, now stated as decisions:

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + TypeScript + Vite | PRD-stated. Vite for a fast local loop. |
| UI state | TanStack Query + Supabase JS client | Cache + retry for free; no bespoke fetch layer |
| Charts | Recharts | Accessible SVG output, keyboard-navigable, no canvas |
| Backend | Supabase (Postgres 15, Auth, Edge Functions, pg_cron) | Auth + RLS + DB in one; local stack runs in Docker, so local == prod |
| Region | **ap-southeast-2 (Sydney)** | Nearest to three of four data subjects; keeps NZ health data in-region |
| Email | Resend | Simple API, delivery webhooks, generous free tier at this volume |
| Hosting | Netlify | PRD-stated. Static SPA + branch previews |
| PDF | `@react-pdf/renderer`, rendered in an Edge Function | Server-side so the GP report is identical regardless of device. ⚠️ **Verify before building the report feature:** Edge Functions run Deno; react-pdf is Node-oriented with native font handling. Spike this first. Fallbacks: a Node function on Netlify, or client-side render with the authorisation check still done server-side |

**Critical architectural note:** with the Supabase client calling Postgres directly, there is no hand-written API tier. **RLS policies are the security boundary.** A missing policy is a data breach, not a bug. This is why the first build slice ships with an authorisation test before any UI exists (build spec §3.3, test T1).

## 3. Alert flow (the safety path)

```
parent submits reading
   │
   ├─► INSERT INTO readings                    ← transaction commits, user gets success
   │
   ├─► AFTER INSERT trigger: evaluate_alert_rule()
   │      breach?  no  → write app_event(alert_evaluated, fired=false) → done
   │               yes → INSERT alert_deliveries (status='queued', dedupe_key)
   │                     ON CONFLICT (dedupe_key) DO NOTHING        ← no double-send
   │
   └─► pg_cron, every minute → Edge Function `dispatch-alerts`
          ├─ SELECT ... WHERE status='queued' FOR UPDATE SKIP LOCKED
          ├─ POST Resend → store provider_message_id, status='sent'
          ├─ on error → attempts++, status='queued' (retry), or 'failed' after 5
          └─ Resend webhook → Edge Function `email-webhook`
                              → status='delivered' | 'bounced' | 'complained'
```

**Why queue instead of sending inside the trigger:** an email provider outage must never roll back or slow down a parent's reading save. The write succeeds; delivery is retried independently. Failure is recorded, not lost.
