Companion to the PRD. The PRD says **what and why**. This says **how**, at a level a coding agent can build from.

Covers scope, build order, analytics, security, non-functional requirements, tests and acceptance criteria. Architecture is in [`PULSE_HLD.md`](./PULSE_HLD.md); schema and policies in [`PULSE_LLD.md`](./PULSE_LLD.md); why the product is shaped this way in [`PULSE_DECISIONS.md`](./PULSE_DECISIONS.md).

## 1. Constraints

Flat statements. The reasoning behind each, and the alternatives rejected, are in [`PULSE_DECISIONS.md`](./PULSE_DECISIONS.md).

| # | Constraint | What it forces |
|---|---|---|
| C1 | Four known users: two parents (NZ, patients), me (UK), my sister (NZ). No public signup; registration is allow-listed | No growth features, no onboarding funnel, no discovery analytics |
| C2 | Both parents log their own readings, separately | Two patient accounts, two data sets, per-patient authorisation |
| C3 | An out-of-range reading must reach family **by email, without them opening the app** | A server, a persistent store, a rule evaluated on write, an email provider, and delivery tracking. The load-bearing requirement of the system |
| C4 | The server may read plaintext readings. No end-to-end encryption | Alert evaluation, charts and reports run server-side. Security posture is encryption in transit + at rest + row-level authorisation |
| C5 | Alert when **systolic > 150**. Diastolic is recorded and charted but never triggers an alert | [LLD §6](./PULSE_LLD.md), and systolic-only display bands |
| C6 | Production starts empty | Historical readings are local fixtures only — see the repo [README](../README.md) |
| C7 | Production-ready means: each user authenticates by email; stored data is secure; health-data security concerns are addressed | §7 is a first-class deliverable, not an appendix |
| C8 | Medication tracking, voice input and logging nudges are out of MVP | No story, no data model, no scheduler |

## 2. What this product is, restated from fundamentals

> A four-person, invite-only web app whose job is to make a daily blood-pressure habit easy for two elderly parents in New Zealand, and to make sure a dangerous reading reaches their daughter in the UK the same day — without her having to go looking for it.

Two things must never fail:

1. **A reading is never lost.** It is written, durably, or the user is told it wasn't.
2. **An alert is never silently dropped.** If an email fails, that failure is visible.

Everything else is a convenience.

## 3. Scope

### 3.1 In (MVP)

- Email registration + authentication, allow-listed to the four known addresses
- Log a reading: date, AM/PM slot, systolic, diastolic, pulse, optional note
- Home screen: most recent reading, prominent, colour-coded
- Trend chart: systolic + diastolic + pulse, 7d / 30d / all
- Family dashboard: My sister and I view either parent's readings and trends
- Threshold alert: email to family viewers when a reading breaches the GP rule
- Alert delivery tracking, visible when it fails
- GP report: PDF export with chart, summary stats, date range

### 3.2 Out (explicitly)

Medication tracking and reminders · BLE/device sync · voice input · native mobile apps · provider portal / EHR · multi-language · public signup · social features · third-party analytics vendors

### 3.3 Build order

Sequencing, not scope reduction. Everything in §3.1 still ships.

| Slice | Contents | Why here |
|---|---|---|
| **S1** | Local Supabase up, schema + RLS, seed fixtures, one E2E test proving a parent can't read the other parent's rows | Nothing is safe to build on an unenforced data model |
| **S2** | Auth: magic-link sign-in, allow-list, persistent session, profile with timezone | Gate for everything |
| **S3** | Log a reading + home screen | The habit. The primary metric depends only on this |
| **S4** | **Alert path**: rule evaluation → queued delivery → email sent → delivery status recorded | The safety case. Highest risk, so earliest |
| **S5** | Trend chart + family dashboard | Read-side |
| **S6** | GP report PDF | Depends on S1–S5; genuinely additive work |
| **S7** | Deploy to cloud, smoke test, handover to parents | [RUNBOOK](./PULSE_RUNBOOK.md) |

## 4. Product analytics

### 4.1 First principle

At n=4, analytics are not for discovering user behaviour. You can phone the users. Analytics exist here for exactly one reason: **to answer a question you cannot answer by asking.**

Applying that filter to the PRD's metrics:

| PRD metric | Verdict |
|---|---|
| Daily active usage rate (90%+) | **Keep.** Genuinely measurable, and it's the metric the product exists to move. |
| Family adoption (100%, Day 1) | Keep as a **launch checklist item**, not a metric. It's a boolean you control. |
| GP sharing (100%) | Keep as a **manual note**. Either the visit happened or it didn't; no instrumentation. |
| Medication adherence | Dropped — feature is out of MVP. |
| Satisfaction survey | Dropped — you'll be having dinner with the respondents. |

### 4.2 The key structural insight

**The readings table is the event log.** Logging consistency needs no analytics infrastructure at all — it is a `GROUP BY` over data you already store. Do not install an analytics SDK to count something the primary table already knows.

```sql
-- Logging consistency, per patient, last 30 local days
select
  p.full_name,
  count(distinct r.reading_local_date)                       as days_with_reading,
  round(100.0 * count(distinct r.reading_local_date) / 30, 1) as pct_days
from profiles p
left join readings r
  on r.patient_id = p.id
 and r.reading_local_date > (now() at time zone p.timezone)::date - 30
where p.role = 'patient'
group by p.full_name;
```

### 4.3 What actually needs instrumenting

Only two things, both operational rather than behavioural:

| Question | Source | Why it can't be answered by asking |
|---|---|---|
| Did the alert email actually arrive? | `alert_deliveries.status`, fed by provider webhook | Silent spam-folder failure is invisible to everyone, including the recipient |
| Is logging consistency drifting down? | `readings`, query above | A slow decline over weeks isn't noticeable in conversation |

### 4.4 The events table

One table, minimal, **no third-party vendor**. Health data never leaves infrastructure you control (C4, §7).

Events to record — that is the whole list:

| Event | Props | Purpose |
|---|---|---|
| `reading_saved` | `patient_id`, `slot`, `via` (`form`) | Redundant with `readings`; kept for save-failure ratio |
| `reading_save_failed` | `patient_id`, `error_code` | Silent write failure is failure mode #1 (§2) |
| `alert_evaluated` | `reading_id`, `fired` (bool) | Proves the rule ran on every write |
| `alert_dispatch_failed` | `delivery_id`, `error_code` | Failure mode #2 |
| `auth_session_expired` | `user_id` | If a parent gets logged out, the habit breaks (LLD §11) |

Deliberately **not** recorded: page views, click paths, time-on-screen, scroll depth, feature usage by person. They would answer questions you can answer at dinner, and they'd mean profiling your parents' behaviour in their own health record.

### 4.5 Surfacing it

One `/health` route, visible to family accounts only:

- Days logged in last 7 / 30, per parent
- Last reading time, per parent, in **their** local time
- Last alert: fired at, delivered / bounced / failed
- Count of `reading_save_failed` and `alert_dispatch_failed` in last 7 days — **non-zero is a bug, investigate**

No dashboards beyond this. If you want a number that isn't here, add it deliberately.

## 5. High-level design

> **Moved.** See [`PULSE_HLD.md`](./PULSE_HLD.md) — context diagram, stack decisions and the alert flow. Edit it there, not here.

## 6. Low-level design

> **Moved.** The low-level design now lives in its own file for the GitHub repo: [`PULSE_LLD.md`](./PULSE_LLD.md).
>
> It covers: schema · RLS policies · profile provisioning · audit log · timezone handling · the alert rule and its test table · alert trigger and dedupe · re-logging the same slot · Edge Functions · frontend module map · session persistence · accessibility.
>
> Edit it there, not here — this section is a pointer to avoid two copies drifting apart. References below point at that document by its own section numbers (LLD §1–§12).

## 7. Security & regulatory

### 7.1 Threat model

Being defended against, in priority order:

1. **Leaked database credentials / anon key** → mitigated by RLS being the boundary. The anon key alone grants nothing.
2. **One family member seeing data they shouldn't** (e.g. Mum seeing Dad's readings) → `care_links` + RLS, with explicit tests.
3. **Account takeover via email** → magic link, short link TTL, allow-listed addresses.
4. **Accidental public exposure** → no public signup; `SELECT` denied by default on every table.
5. **Third-party script exfiltration** → no third-party analytics or tag managers. Strict CSP.

**Not** defended against, deliberately: a compromised hosting or database provider. C4 makes this an explicit, accepted trade in exchange for server-side alerting.

### 7.2 Controls

| Control | Implementation |
|---|---|
| Transport | HTTPS only, HSTS |
| At rest | Supabase/AWS AES-256 disk encryption |
| Authorisation | RLS on every table; default deny; `security definer` helpers audited |
| Registration | Allow-list of four email addresses; signup otherwise disabled |
| Secrets | Service-role key only in Edge Function env. **Never** in frontend code or the repo |
| CSP | `default-src 'self'`; no inline scripts; no third-party origins |
| Audit | `audit_log` on reading insert/update/delete and care-link changes |
| Backups | Supabase daily PITR; plus monthly manual CSV export to encrypted storage |
| Dependencies | `npm audit` in CI; Dependabot on |

### 7.3 Regulatory position

*Not legal advice. This is a stated position to be checked, not a compliance certificate.*

- Data subjects: three in New Zealand, one in the UK. **NZ Privacy Act 2020** is the primary frame; **UK GDPR** is potentially in scope via the UK-based family member.
- **Position:** this is a four-person family tool with no commercial purpose and no third-party data sharing. Both UK GDPR (Art. 2(2)(c)) and the NZ Privacy Act contain exemptions for purely personal, domestic or household activity. This app is a strong candidate for both.
- **That exemption weakens immediately** if the app is ever offered to anyone outside the family, or if the GP is given an account rather than a PDF. If that happens, this section must be redone properly — DPIA, privacy notice, DSAR process, retention policy, lawful basis for processing special-category health data.
- **Regardless of exemption**, the engineering standards in §7.2 apply in full. The exemption saves paperwork, not care.
- Data residency: **ap-southeast-2 (Sydney)**, nearest region to three of four data subjects. Noted that this is Australia, not NZ — Supabase has no NZ region.
- The GP receives a **PDF that a family member sends**. The GP gets no account and no access. This keeps the data sharing boundary at the family line.

## 8. Non-functional requirements

| # | Requirement | How verified |
|---|---|---|
| N1 | First contentful paint < 2s on 4G, mid-range tablet | Lighthouse in CI |
| N2 | Reading save → success feedback < 1s p95 | Instrumented timing in E2E test |
| N3 | Chart renders 30 days < 1s | Performance test with fixture data |
| N4 | Alert email sent within **5 minutes** of a breaching reading | E2E test against local mail catcher |
| N5 | A failed alert delivery is visible in the UI within 1 hour | §4.5 health view + webhook |
| N6 | Works offline for *viewing* the last-loaded reading | Service worker, read-only cache |
| N7 | No reading is lost on a failed save — the user is told, and input is preserved | Test with network disabled |
| N8 | WCAG 2.1 AA | axe-core in CI + manual keyboard/screen-reader pass |

> **N6 scope note:** offline *writing* is deliberately out. Queued offline writes create conflict-resolution and stale-alert problems that aren't worth it for a user at home on wifi. If this proves wrong in real use, revisit — it's a real design decision, not an oversight.

## 9. Local development, testing, deployment

### 9.1–9.2 Local stack and fixtures

> **Moved.** See the repo [`README.md`](../README.md) — local setup, running the stack, and how fixtures differ from production (which starts empty).

### 9.3 Test plan

| ID | Test | Type | Slice |
|---|---|---|---|
| T1 | Mum's session cannot read Dad's readings (direct API call, not UI) | Integration, RLS | S1 |
| T2 | A family viewer without a `care_link` reads nothing | Integration, RLS | S1 |
| T3 | Anon key with no session reads nothing from any table | Integration, RLS | S1 |
| T4 | Every row of the [LLD §6](./PULSE_LLD.md) test table, in **both** SQL and `lib/bp.ts`. Include `140/95 → no alert` explicitly — it is the row most likely to be "helpfully" broken by an implementer who assumes diastolic matters | Unit | S4 |
| T5 | Breaching reading → `alert_deliveries` row → email in Inbucket < 5 min | E2E | S4 |
| T6 | Same slot re-saved with the same systolic → exactly one email | E2E | S4 |
| T6b | Non-breaching reading corrected upward into breach → alert sent | E2E | S4 |
| T7 | Resend failure → retried, then `failed`, then visible in §4.5 | Integration | S4 |
| T8 | NZ reading at 11:30pm NZDT shows the correct local date for a UK viewer | Unit, timezone | S3 |
| T9 | Save with network offline → error shown, input preserved | E2E | S3 |
| T10 | Log a reading in ≤3 interactions from home screen | E2E | S3 |
| T11 | axe-core clean; keyboard-only journey completes | A11y | S5 |
| T12 | Refresh-token lifetime is configured to 90 days, and a clock-mocked 30-day-idle session refreshes silently without a sign-in prompt | Config assertion + unit | S2 |

**T1 is the gate.** Don't build UI until it passes.

### 9.4 Local → cloud

> **Moved.** See [`PULSE_RUNBOOK.md`](./PULSE_RUNBOOK.md) — first deploy, post-deploy checks, alert troubleshooting and rollback.

---

## 10. Acceptance criteria

Phrased as assertions. PRD story → testable outcome.

**Story 1 — log a reading**
- [ ] From home, a logged-in patient reaches a saved reading in ≤ 3 interactions
- [ ] Systolic, diastolic, pulse fields are ≥ 18pt with ≥ 44pt targets, `inputmode="numeric"`
- [ ] AM/PM is one tap, defaulting to the current period in the **patient's** timezone
- [ ] Out-of-range input is rejected with plain-language text, not a code
- [ ] A confirmation step shows the values before saving
- [ ] Success feedback is visible and does not auto-dismiss in under 3 seconds
- [ ] On save failure the user is told and the entered values are preserved (N7)

**Story 2 — trends**
- [ ] Chart shows systolic, diastolic and pulse across 7d / 30d / all
- [ ] Normal / elevated / high bands per LLD §6, conveyed by colour **and** text
- [ ] Axis labels and legend readable at 200% zoom
- [ ] 30 days renders in < 1s (N3)
- [ ] Chart data is reachable without a mouse

**Story 3 — family dashboard + alerts**
- [ ] A family viewer sees only patients they have a `care_link` to (T1, T2)
- [ ] Latest reading per parent, with patient-local timestamp and zone label
- [ ] A breaching reading produces an email within 5 minutes (N4, T5)
- [ ] The email contains the reading, the band, the threshold breached, and patient-local time
- [ ] One alert per patient / local day / slot / recipient (T6)
- [ ] A failed delivery is visible in the health view (N5, T7)
- [ ] A patient can revoke a viewer's access, and it takes effect immediately

**Story 4 — GP report**
- [ ] Date range selection; PDF contains chart, table, and summary stats (mean, max, min, % of readings in each band, days logged)
- [ ] Generated server-side; authorisation re-checked, client input not trusted
- [ ] Patient name, date range and generation date on the page; printable A4

**Production readiness (C7)**
- [ ] All four users registered and authenticated by email; no one else can register
- [ ] Every table has RLS enabled with a default-deny posture
- [ ] No service-role key in any client bundle (grep the build output)
- [ ] SPF/DKIM/DMARC pass; production smoke test email lands in the inbox, not spam
- [ ] Backups confirmed restorable at least once before handover

---

## 11. Decisions and open vetoes

> **Moved.** Reasoning, rejected alternatives, resolved questions and the open vetoes live in [`PULSE_DECISIONS.md`](./PULSE_DECISIONS.md).

**Two spikes to run during the build**, carried here because they are build tasks, not decisions:

- `@react-pdf/renderer` under Deno — verify before building the GP report (HLD §2). Fallbacks: a Node function on Netlify, or client-side render with the authorisation check still server-side.
- `pg_net` enabled alongside `pg_cron`, so the scheduled job can call the Edge Function (LLD §9).

## 12. Phase 2 candidates

Not scoped. Recorded so they aren't re-litigated: daily logging nudges · medication tracking · offline write queue · BLE device sync · shared household device profiles · streak display for motivation.
