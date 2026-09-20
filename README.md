# Pulse-App
A blood pressure web app for elderly adults

Its job is to make daily logging easy for two elderly parents, and to make sure a reading above the GP's threshold reaches family the same day — by email, without anyone having to open the app.

## Documentation

| Document                                         | What's in it                                                                           |
| ------------------------------------------------ | -------------------------------------------------------------------------------------- |
| [`docs/PULSE_PRD.md`](docs/PULSE_PRD.md)         | What and why — problem, users, goals, stories                                          |
| [`docs/PULSE_HLD.md`](docs/PULSE_HLD.md)         | Architecture — context, stack, alert flow                                              |
| [`docs/PULSE_LLD.md`](docs/PULSE_LLD.md)         | Schema, RLS policies, triggers, Edge Functions, module map                             |
| [`docs/PULSE_BUILD_SPEC.md`](docs/PULSE_SPEC.md)       | Build spec — scope, build order, analytics, security, NFRs, tests, acceptance criteria |
| [`docs/PULSE_RUNBOOK.md`](docs/PULSE_RUNBOOK.md) | Deploying, post-deploy checks, troubleshooting                                         |

**Start here if you're building:** `docs/PULSE_BUILD_SPEC.md` §3.3 (build order), then the LLD.

## Stack

React + TypeScript (Vite) · Supabase (Postgres 15, Auth, Edge Functions, `pg_cron`, `pg_net`) · Resend · Netlify.

There is **no hand-written API tier.** The frontend talks to Postgres through PostgREST, and Row Level Security policies are the security boundary — a missing policy is a data breach, not a bug. See LLD §2.

## Running it locally

```bash
npm create vite@latest pulse -- --template react-ts
npx supabase init && npx supabase start   # Postgres, Auth, Studio, Inbucket (mail catcher)
npx supabase db reset                     # applies migrations + seed
npm run dev
```

Local Supabase runs the same Postgres and the same RLS as production. **Inbucket catches alert emails locally**, so the entire alert path (HLD §3) is testable end-to-end on your laptop without sending a real email.

## Fixtures vs production data

**Production starts empty** — no historical readings are migrated. That is about **production**. Local development needs realistic data:

- `supabase/seed.sql` — two patient profiles, two family profiles, care links, alert rules, and several months of readings.
- Generate the readings from the historical markdown in the vault (`Blood-Pressure-Data-*.md`) using the existing `scripts/bp_parser.py` — it already parses the `DD/MM/YYYY | AM/PM | 137/87 - 71` format. Write a one-off `scripts/seed_from_markdown.py` that emits `seed.sql`.
- The fixture must include **at least one breaching reading** so the alert path is exercised on every `db reset`.
- `seed.sql` is local-only and must never run against production.

## Testing

```bash
npm test          # unit
npm run test:e2e  # end-to-end, against the local Supabase stack
```

The test plan is in `docs/PULSE_BUILD_SPEC.md` §9.3. **T1 — "one patient's session cannot read the other patient's readings" — is the gate.** Don't build UI until it passes.
