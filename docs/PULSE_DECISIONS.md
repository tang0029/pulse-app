# Pulse — Decision Log

Why the product is shaped the way it is. Append-only: add new decisions at the bottom with a date; don't edit old ones, supersede them.

This exists so decisions aren't silently re-litigated — or silently undone during a build. The **requirements** these produced live in `PULSE_SPEC.md`; this document holds the reasoning and what was rejected.

---

## 2026-09-20 — Foundational decisions

Settled in a first-principles session against the PRD.

### D1 — This is a four-person family tool, not a market product

**Users:** two parents (NZ, patients), me (UK), my sister (NZ). No public signup; registration is allow-listed.

**Rejected:** treating the PRD's competitive framing as the design target. Eight-competitor market gaps justified *building* it, but they don't shape a product with four known users.

**Consequences:** no growth features, no onboarding funnel, no discovery analytics, no tenancy model.

### D2 — Alerting is a safety requirement, not a product feature

An out-of-range reading must reach a family viewer **the same day, by email, without them opening the app**.

**Why it matters:** the failure mode this product exists to prevent is a sustained high reading nobody notices until there's a health event. A dashboard you have to remember to check doesn't prevent it.

**Consequences:** forces a server, a persistent store, a rule evaluated on every write, a transactional email provider, and delivery tracking. This single requirement is why the architecture isn't a browser-only app.

### D3 — The server reads plaintext; end-to-end encryption rejected

**Why:** a server that cannot read a systolic value cannot compare it to a threshold, cannot decide to send an email, and cannot put the number in it. E2EE and D2 are mutually exclusive.

**Accepted trade:** a compromised hosting or database provider is explicitly *not* defended against. In exchange, alerting, server-side charts and the GP report all work.

**Replaced with:** encryption in transit and at rest, plus row-level authorisation — "nobody but the four of us can read this," which is the property actually wanted.

### D4 — Alert threshold is systolic > 150; diastolic is ignored

Per the GP's recommendation for these patients. `151/80` alerts. `145/80` and `140/95` do not.

**Consequence:** display bands are systolic-only too, so a reading shown in red and an email being sent are always the same condition. Diastolic and pulse are still recorded and charted in full.

**Open veto:** if you'd rather diastolic still coloured the UI while staying out of the alert rule, say so — but accept that red-on-screen would then sometimes mean no email.

### D5 — Most of the PRD's metrics were dropped

At n=4, metrics can't discover behaviour — you can phone the users.

- **Kept:** daily logging consistency. It measures the habit the product exists to build, and it needs no instrumentation because the readings table *is* the event log.
- **Kept as launch checklist, not metrics:** family adoption, GP sharing. Booleans you control.
- **Dropped:** the satisfaction survey (worse than asking at dinner), data accuracy vs device readings (no device integration in MVP), medication adherence (feature out of scope).

**Rejected:** behavioural telemetry — page views, click paths, feature usage. It would answer questions answerable in conversation, and it would mean profiling elderly parents' behaviour inside their own health record. Five operational events are recorded instead.

### D6 — Production starts empty

No migration of the historical readings (Dec 2025 – Jul 2026) held in markdown.

**Note:** this is about *production*. Local development seeds fixtures from that historical data via `scripts/bp_parser.py`, including at least one breaching reading so the alert path is exercised on every reset. **Open veto:** say so if you'd rather develop against synthetic data.

### D7 — "Production-ready" defined

(a) Each user registers and authenticates by email; (b) stored data is secure; (c) health-data security concerns are addressed.

**Consequence:** security is a first-class section of the spec with its own threat model, not a checklist appendix. Two things must never fail: a reading is never lost, and an alert is never silently dropped.

### D8 — Medication tracking, voice input and nudges are out of MVP

- **Medication:** removed from the PRD's goals and metrics by the owner. No story, no data model.
- **Voice input:** removed by the owner — largest technical lift in the MVP, deferred to a later iteration.
- **Daily logging nudges:** parked. Nudging is a live goal, but consistency is already measurable without building the nudge.

### D9 — Feature-prioritisation telemetry not built

"What to build next" was named as a live question, then parked.

**Position for the record:** with four users and one builder, prioritisation telemetry is not justifiable. Ask your sister.

---

## 2026-09-20 — Frontend review

### D10 — React + TypeScript + Vite confirmed, with three corrections

Reviewed against the alternatives rather than inherited from the PRD.

**Confirmed, because:** once RLS is the security boundary there is no API tier, so a **static SPA** is the right shape. **Rejected: Next.js / Remix** — they add a server this design doesn't need and invite server-side access via the service role, moving authorisation out of RLS into hand-written route code. **Rejected: no framework / vanilla** — ten screens of forms and charts don't strictly need React, but session handling, route guards, async states and charts would all be hand-rolled, and React + Supabase is the best-documented path for a solo build with a coding agent.

**TypeScript earns its place** via `supabase gen types typescript` — types generated from the schema, so a column rename breaks the build everywhere it's now wrong (LLD §10).

**Three corrections applied:**

1. **Recharts was described as "keyboard-navigable". That was wrong** — its accessibility layer is opt-in and varies by chart type. Every chart now ships with a visible data table beneath it, and the table, not the chart, is the accessible path.
2. **SPA route changes break screen-reader navigation** and `axe-core` cannot detect it. Focus management and a live-region announcement on every route change are now a stated requirement (LLD §12, test T11).
3. **TanStack Query scoped down** — kept for retry behaviour (N7) and the offline read cache (N6), not for optimistic updates or background refetching.

**Not changed:** no Preact swap. The bundle saving doesn't justify the complexity for users at home on wifi.

---

## 2026-09-20 — Missed-reading alerts

### D11 — Absence of readings now raises an alert (supersedes part of D8)

**The gap:** every alert in the design fired on a reading that existed. A parent who stopped logging entirely — unwell, hospitalised, unable to manage the app — produced no high readings and therefore no alerts. The system was silent in one of the states most worth knowing about, and the `/health` view only helps someone who opens it, which is the assumption the whole email design rejects.

**Decision:** if a patient logs nothing for `missed_days` consecutive local days, email their family viewers. In MVP, not Phase 2.

**What this supersedes:** D8 parked *nudges*, and that still stands — a reminder **to the parent** remains out of scope. This is a different thing: an alert **to family**, justified as safety rather than engagement. The two were conflated, and separating them is what reopened the decision.

**Design points that matter:**

- **One email per absence episode, not per day.** The dedupe key is built from the day after the last reading, so a week of silence sends one email and re-arms the moment a reading arrives. Daily nagging gets filtered, and a filtered alert is a dead alert.
- **Reuses the existing queue, dispatcher, retry and webhook.** One delivery mechanism, two templates — not a second alerting system.
- **Evaluated in the patient's local evening**, not on server time.
- **Default window: 2 days**, chosen so one skipped morning doesn't fire. ⚠️ **Open — confirm the right window.** It is one column value, changeable without a migration.

---

## Resolved questions

| # | Question | Resolution |
|---|---|---|
| Q1 | What is the alert threshold? | Systolic > 150. Diastolic ignored — D4 |
| Q2 | Same threshold for both parents? | Yes. Per-patient rows still allow divergence later |
| Q3 | Re-alert on a correction that breaches? | Yes. `dedupe_key` includes the systolic value; the trigger fires on UPDATE. LLD §7 |
| Q4 | Does the sister receive alerts? | Yes. Both family viewers, `alerts_on = true` |
| Q5 | Do parents see each other's readings? | No. Each parent sees only their own; both family viewers see both parents |
| Q6 | Daily logging nudge in MVP? | Out — D8 |
| Q7 | Medication goal in the PRD? | Removed by the owner — D8 |

## Open vetoes

Decisions made on your behalf that are cheap to reverse. Reverse them by saying so.

1. **Display bands are systolic-only** (D4) — the alternative is diastolic colouring the UI while never alerting.
2. **Local fixtures come from the historical vault readings** (D6) — the alternative is synthetic data.
3. **Regulatory approach is a stated position, not a compliance programme** (`PULSE_SPEC.md` §7) — the household-activity exemption under UK GDPR and the NZ Privacy Act is argued as an assumption to check. The alternative is full DPIA, privacy notice, DSAR process and retention policy, which is a materially bigger piece of work.

---

## Superseded documents

The Obsidian vault holds earlier planning docs that contradict this product. They are superseded; do not build from them.

| Document | Why superseded |
|---|---|
| `docs/PRD.md` | Older vault PRD — mobile app, medication in MVP, different metrics |
| `Web-BP-Tracker-Brief-19-04-2026.md` | No backend, localStorage only, caregiver sharing out of scope |
| `docs/Pulse-Design-System.md` | Revised 2026-09-20 to match this product. Its visual foundations remain the reference; its earlier scope, ≥160 threshold and localStorage prototype were wrong |
