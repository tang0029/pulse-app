# Pulse — Low-Level Design

Implementation-level design for **Pulse**, a blood pressure tracking web app for a single family: two patients in New Zealand, two family viewers (one in the UK, one in NZ).

- **What and why:** [`bloodpressure_prd.md`](./bloodpressure_prd.md)
- **Analytics, high-level design, security, test plan, deploy runbook:** the build spec (`SPEC.md`)
- **This document:** schema, authorisation, the alert rule, triggers, functions, frontend structure

## Context this design assumes

| Fact | Consequence for this design |
|---|---|
| Four known users; no public signup | Registration is allow-listed; no tenancy or org model |
| Both parents log independently | Two patient accounts, per-patient authorisation |
| An out-of-range reading must reach a family viewer **by email**, without them opening the app | A server, a rule evaluated on write, a delivery queue, and delivery tracking |
| The server is permitted to read plaintext readings | End-to-end encryption rejected: a server that can't read a systolic value can't alert on it |
| GP rule: **alert when systolic > 150**; diastolic is not a clinical concern for these patients | §6 |
| Patients are in `Pacific/Auckland`; one viewer is in `Europe/London` | §5 — the hardest correctness problem here |
| Stack: React + TypeScript + Vite; Supabase (Postgres 15, Auth, Edge Functions, `pg_cron`, `pg_net`); Resend for email | Below |

**Architectural note that governs everything in §2:** the Supabase client talks to Postgres directly. There is no hand-written API tier. **Row Level Security policies are the security boundary** — a missing policy is a data breach, not a bug.

Two invariants the whole design serves:

1. A reading is never lost — it is written durably, or the user is told it wasn't.
2. An alert is never silently dropped — if an email fails, that failure is visible.

---
## 1. Schema

```sql
create type user_role   as enum ('patient', 'family');
create type slot_type   as enum ('am', 'pm');
create type delivery_status as enum ('queued','sent','delivered','bounced','complained','failed');

-- Profiles ------------------------------------------------------------------
create table profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  full_name   text        not null,
  role        user_role   not null,
  timezone    text        not null default 'Pacific/Auckland',  -- IANA
  created_at  timestamptz not null default now()
);

-- Who may see whom ----------------------------------------------------------
create table care_links (
  id          uuid primary key default gen_random_uuid(),
  patient_id  uuid not null references profiles(id) on delete cascade,
  viewer_id   uuid not null references profiles(id) on delete cascade,
  alerts_on   boolean not null default true,   -- viewer receives threshold emails
  created_at  timestamptz not null default now(),
  unique (patient_id, viewer_id),
  check (patient_id <> viewer_id)
);

-- Readings ------------------------------------------------------------------
create table readings (
  id                 uuid primary key default gen_random_uuid(),
  patient_id         uuid not null references profiles(id) on delete cascade,
  recorded_at        timestamptz not null,
  recorded_tz        text        not null,   -- IANA tz at time of writing
  reading_local_date date        not null,   -- recorded_at in recorded_tz
  slot               slot_type   not null,
  systolic           smallint    not null check (systolic  between 60 and 260),
  diastolic          smallint    not null check (diastolic between 30 and 180),
  pulse              smallint        null check (pulse     between 30 and 220),
  note               text            null check (char_length(note) <= 500),
  created_by         uuid not null references profiles(id),
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),
  check (systolic > diastolic),
  unique (patient_id, reading_local_date, slot)   -- one AM + one PM per local day
);
create index on readings (patient_id, recorded_at desc);

-- Threshold rules -----------------------------------------------------------
create table alert_rules (
  id            uuid primary key default gen_random_uuid(),
  patient_id    uuid not null references profiles(id) on delete cascade,
  systolic_max  smallint not null default 150,   -- alert when systolic  > this
  diastolic_max smallint not null default 90,    -- alert when diastolic > this
  active        boolean  not null default true,
  source_note   text,          -- e.g. "GP recommendation, Sept 2026"
  updated_at    timestamptz not null default now()
);

-- Alert delivery ------------------------------------------------------------
create table alert_deliveries (
  id                  uuid primary key default gen_random_uuid(),
  reading_id          uuid not null references readings(id) on delete cascade,
  rule_id             uuid not null references alert_rules(id),
  recipient_id        uuid not null references profiles(id),
  recipient_email     text not null,
  dedupe_key          text not null unique,   -- patient|local_date|slot|recipient
  status              delivery_status not null default 'queued',
  attempts            smallint not null default 0,
  provider_message_id text,
  last_error          text,
  created_at          timestamptz not null default now(),
  sent_at             timestamptz,
  updated_at          timestamptz not null default now()
);
create index on alert_deliveries (status) where status = 'queued';

-- Audit + analytics ---------------------------------------------------------
create table audit_log (
  id         bigserial primary key,
  actor_id   uuid references profiles(id),
  action     text not null,      -- reading.insert | reading.update | care_link.grant | ...
  entity     text not null,
  entity_id  uuid,
  meta       jsonb not null default '{}',
  at         timestamptz not null default now()
);

create table app_events (
  id          bigserial primary key,
  user_id     uuid references profiles(id),
  event_name  text not null,
  props       jsonb not null default '{}',
  occurred_at timestamptz not null default now()
);
```

## 2. Row Level Security

**This is the security model. Nothing else enforces it.**

```sql
alter table profiles         enable row level security;
alter table readings         enable row level security;
alter table care_links       enable row level security;
alter table alert_rules      enable row level security;
alter table alert_deliveries enable row level security;
alter table app_events       enable row level security;
alter table audit_log        enable row level security;

-- helper: may the current user view this patient?
create or replace function can_view_patient(p uuid)
returns boolean language sql security definer stable as $$
  select p = auth.uid()
      or exists (select 1 from care_links
                  where patient_id = p and viewer_id = auth.uid());
$$;

-- profiles: self, plus anyone you're linked to (either direction)
create policy profiles_select on profiles for select
  using (id = auth.uid() or can_view_patient(id)
         or exists (select 1 from care_links
                     where viewer_id = profiles.id and patient_id = auth.uid()));
create policy profiles_update on profiles for update
  using (id = auth.uid()) with check (id = auth.uid());

-- readings: patient sees own; linked viewers see theirs; ONLY the patient writes
create policy readings_select on readings for select
  using (can_view_patient(patient_id));
create policy readings_insert on readings for insert
  with check (patient_id = auth.uid() and created_by = auth.uid());
create policy readings_update on readings for update
  using (patient_id = auth.uid()) with check (patient_id = auth.uid());
create policy readings_delete on readings for delete
  using (patient_id = auth.uid());

-- care_links: the PATIENT controls who sees their data
create policy care_links_select on care_links for select
  using (patient_id = auth.uid() or viewer_id = auth.uid());
create policy care_links_all on care_links for all
  using (patient_id = auth.uid()) with check (patient_id = auth.uid());

-- alert_rules: patient manages own; viewers may read
create policy alert_rules_select on alert_rules for select
  using (can_view_patient(patient_id));
create policy alert_rules_write on alert_rules for all
  using (patient_id = auth.uid()) with check (patient_id = auth.uid());

-- deliveries: recipient or patient may read; only service role writes
create policy deliveries_select on alert_deliveries for select
  using (recipient_id = auth.uid()
         or exists (select 1 from readings r
                     where r.id = reading_id and r.patient_id = auth.uid()));

-- app_events: clients may INSERT their own events, never read them.
-- Required: reading_save_failed and auth_session_expired are only observable
-- client-side, and they are the two invariants named at the top of this doc.
create policy app_events_insert on app_events for insert
  with check (user_id = auth.uid());
-- (no select policy: default deny)

-- audit_log: no client access at all. Written by trigger (security definer).
```

## 3. Profile provisioning

Nothing creates a `profiles` row on its own. `auth.users` is managed by Supabase Auth; `profiles` is ours.

```sql
create or replace function handle_new_user()
returns trigger language plpgsql security definer as $$
begin
  insert into profiles (id, full_name, role, timezone)
  values (new.id,
          coalesce(new.raw_user_meta_data->>'full_name', new.email),
          coalesce((new.raw_user_meta_data->>'role')::user_role, 'patient'),
          coalesce(new.raw_user_meta_data->>'timezone', 'Pacific/Auckland'));
  return new;
end $$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function handle_new_user();
```

"Invite manually" (the build spec's deploy runbook (step 8)) means: invite each of the four addresses from the Supabase dashboard with `full_name`, `role` and `timezone` set in user metadata — `Pacific/Auckland` for the three NZ accounts, `Europe/London` for Tiara. Then insert by hand in `supabase/bootstrap.sql` (run once, never as a migration):
- **4 `care_links` rows** — (Dad → Tiara), (Dad → Sister), (Mum → Tiara), (Mum → Sister), all `alerts_on = true`. No parent-to-parent link, which is what keeps their readings private from each other (Q5).
- **2 `alert_rules` rows** — one per parent, `systolic_max = 150`, `source_note = 'GP recommendation, Sept 2026 — systolic only'`.

## 4. Audit log

The build spec's security section claims an audit control. This is what implements it — a named control with no mechanism is worse than no claim.

```sql
create or replace function write_audit()
returns trigger language plpgsql security definer as $$
begin
  insert into audit_log (actor_id, action, entity, entity_id, meta)
  values (auth.uid(),
          tg_table_name || '.' || lower(tg_op),
          tg_table_name,
          coalesce(new.id, old.id),
          case when tg_op = 'DELETE' then to_jsonb(old) else to_jsonb(new) end);
  return coalesce(new, old);
end $$;

create trigger trg_audit_readings
  after insert or update or delete on readings
  for each row execute function write_audit();

create trigger trg_audit_care_links
  after insert or update or delete on care_links
  for each row execute function write_audit();
```

> **Mum must not see Dad's readings.** That follows from `can_view_patient` returning false without a `care_links` row. It is the single most important assertion in the test suite (build spec, test T1) — write that test before any UI.

## 5. Timezone handling

The hardest correctness problem in this app, and it arises directly from F1: **patients are in NZ, one viewer is in the UK, 11–13 hours apart depending on DST.**

Rules, applied without exception:

1. `recorded_at` is `timestamptz` — an absolute instant. Never `timestamp`.
2. A reading's **local date and AM/PM slot belong to the patient**, not the viewer or the server. `reading_local_date` is computed at write time as `(recorded_at at time zone profiles.timezone)::date` and **stored**, because the unique constraint on one-AM-one-PM-per-day must be evaluated in the patient's frame.
3. `recorded_tz` is stored alongside so a reading remains interpretable if a profile timezone is ever changed.
4. **Every timestamp rendered to any user shows the patient's local time, with the zone labelled**: `8:05am, Tue 22 Sep (NZDT)`. Tiara reading an alert in London must not have to compute the offset.
5. "Today" on the parent's home screen means today in `Pacific/Auckland`. On the family dashboard it *also* means today in Auckland. One reference clock: the patient's.
6. The AM/PM slot is a user choice, not derived from the clock — a reading taken at 12:30pm may legitimately be the morning reading.

```sql
create or replace function set_reading_local_fields()
returns trigger language plpgsql as $$
declare tz text;
begin
  select timezone into tz from profiles where id = new.patient_id;
  new.recorded_tz        := coalesce(new.recorded_tz, tz);
  new.reading_local_date := (new.recorded_at at time zone new.recorded_tz)::date;
  new.updated_at         := now();
  return new;
end $$;

create trigger trg_reading_local
  before insert or update on readings
  for each row execute function set_reading_local_fields();
```

## 6. The alert rule

From F5. **A reading is concerning when systolic > 150. Diastolic is ignored.**

Per the GP, diastolic is not a clinical concern for these patients. It is still captured, stored and charted — it simply never fires an alert.

```sql
create or replace function is_concerning(sys smallint, sys_max smallint)
returns boolean language sql immutable as $$
  select sys > sys_max;
$$;
```

Test table — implement these exactly (default `sys_max` = 150):

| systolic | diastolic | Expected | Source |
|---|---|---|---|
| 151 | 80 | **alert** | GP rule (F5) |
| 150 | 90 | no alert | "greater than 150" — strictly greater |
| 145 | 80 | no alert | GP rule (F5) |
| 140 | 95 | no alert | Diastolic ignored by instruction |
| 168 | 95 | **alert** | Systolic breach |
| 120 | 70 | no alert | Normal |

> `alert_rules.diastolic_max` is retained in the schema but **unused by the rule**. It exists so a future GP instruction can be applied without a migration. Do not read it in application code.

**Display bands** — colour-coding on screen. Aligned to the same systolic-only logic so that *red on screen* and *email sent* always agree. A reading shown in red with no alert would undermine trust in both.

| Band | Condition | Colour |
|---|---|---|
| Normal | sys ≤ 130 | green |
| Elevated | sys 131–150 | amber |
| High | sys > 150 | red — identical to the alert rule |

**Evaluated in order: high → elevated → normal.** First match wins.

Diastolic and pulse are displayed and charted with full prominence, but are not banded. ⚠️ This follows from the GP ignoring diastolic — flag it if you'd rather diastolic still coloured the display while staying out of the alert.

## 7. Alert trigger

```sql
create or replace function evaluate_alert_rule()
returns trigger language plpgsql security definer as $$
declare r alert_rules%rowtype; fired boolean := false; v record;
begin
  select * into r from alert_rules
   where patient_id = new.patient_id and active limit 1;
  if found and is_concerning(new.systolic, r.systolic_max) then
    fired := true;
    for v in
      select p.id, u.email
        from care_links cl
        join profiles p on p.id = cl.viewer_id
        join auth.users u on u.id = p.id
       where cl.patient_id = new.patient_id and cl.alerts_on
    loop
      insert into alert_deliveries (reading_id, rule_id, recipient_id, recipient_email, dedupe_key)
      values (new.id, r.id, v.id, v.email,
              new.patient_id || '|' || new.reading_local_date || '|' || new.slot
                              || '|' || v.id || '|' || new.systolic)
      on conflict (dedupe_key) do nothing;   -- one alert per distinct systolic per slot per recipient
    end loop;
  end if;
  insert into app_events (user_id, event_name, props)
  values (new.patient_id, 'alert_evaluated',
          jsonb_build_object('reading_id', new.id, 'fired', fired));
  return new;
end $$;

create trigger trg_evaluate_alert
  after insert or update on readings
  for each row execute function evaluate_alert_rule();
```

**Dedupe semantics: one alert per patient / local date / slot / recipient / systolic value.** Including the systolic value in the key is what makes corrections work (Q3):

| Scenario | Result |
|---|---|
| 168 logged | Alert sent |
| Same 168 re-saved unchanged | No second email — `dedupe_key` collides |
| 128 logged, corrected to 168 | **Alert sent** — different key, and the correction is the point |
| 168 logged, corrected to 120 | No new alert; the first email already went. Correct behaviour: you were told, and you'll see the corrected value on the dashboard |
| 168 corrected to 172 | Second alert. Rare, and worth knowing about |

The trigger fires on `INSERT OR UPDATE` for exactly this reason.

## 8. Re-logging the same slot

`unique (patient_id, reading_local_date, slot)` means a parent who re-takes their morning reading, or corrects a typo by logging again, hits a unique violation. A raw Postgres `23505` satisfies neither N7 nor Story 1's plain-language requirement.

Required behaviour in `LogReading.tsx`:

1. Before save, check for an existing reading for this patient / local date / slot.
2. If one exists, show: *"You already logged a morning reading today — 142/88. Replace it?"* with **Replace** and **Cancel**. Never a silent overwrite; never a dead end.
3. Replace performs an `UPDATE`, not a delete-and-insert, so `readings.id` is stable and `audit_log` records the change.
4. The unique violation is still handled defensively as a race — same message, no stack trace, input preserved.

Interaction with alerting: replacing a reading re-runs `evaluate_alert_rule` (the trigger fires on `UPDATE`). Because `dedupe_key` includes the systolic value, a correction that newly breaches 150 **does** send an alert, while re-saving the same value does not. See the table in §7.

## 9. Edge Functions

| Function | Trigger | Responsibility |
|---|---|---|
| `dispatch-alerts` | `pg_cron` every minute, POSTing via **`pg_net`** (both extensions must be enabled — deploy runbook step 5) | Claim queued rows `FOR UPDATE SKIP LOCKED`, send via Resend, record `provider_message_id`, set `sent`. On error: `attempts++`, back to `queued`; after 5 attempts → `failed` + `alert_dispatch_failed` event |
| `email-webhook` | Resend POST | Verify signature, map event → `delivered` / `bounced` / `complained` |
| `generate-report` | Called from UI (S6) | Render PDF for `{patient_id, from, to}`. Re-checks `can_view_patient` server-side — never trusts the client |

**Alert email content** — designed to be actionable from a phone lock screen:

- Subject: `Pulse: Dad's reading is 168/95 (Tue 8:05am NZDT)`
- Body: the reading, the band, the threshold it breached, the previous three readings for context, patient-local timestamp with zone, link to the dashboard.
- Never include: the full history, or anything that makes the email itself a health record worth intercepting beyond the single reading.

## 10. Frontend module map

```
src/
├── lib/
│   ├── supabase.ts        # client singleton
│   ├── time.ts            # patient-local formatting. ALL date rendering goes here
│   ├── bp.ts              # isConcerning(), band(), validation ranges — mirrors §6
│   └── analytics.ts       # app_events writer (the 5 events listed in build spec §4.4, nothing more)
├── auth/
│   ├── SignIn.tsx         # email → magic link. One field, one button
│   └── RequireAuth.tsx    # session guard + silent refresh
├── patient/
│   ├── Home.tsx           # latest reading, band, "Log a reading" primary action
│   ├── LogReading.tsx     # form → confirm → save → success (incl. §8 replace flow)
│   ├── Sharing.tsx        # who can see my readings; toggle alerts; revoke access
│   └── Trends.tsx         # 7d / 30d / all
├── family/
│   ├── Dashboard.tsx      # both parents, latest + status
│   ├── PatientDetail.tsx  # one parent: chart + history
│   ├── Health.tsx         # operational view — build spec §4.5
│   └── Report.tsx         # date range → PDF (S6)
└── ui/                    # Button, NumberField, StatusPill, Chart wrappers
```

`lib/bp.ts` duplicates the SQL rule in §6 for instant client-side feedback. **The database is authoritative**; the client copy is a UX affordance. Both are tested against the same table in §6 — if they diverge, the test fails.

## 11. Session persistence

Direct consequence of the 90% daily-logging target. A 78-year-old asked for a password at 8am will stop logging.

- Sign-in: **magic link by email**, once. No password to forget.
- Session: persistent, auto-refreshing, `localStorage`-backed. Refresh token lifetime **90 days**.
- **A patient must never be asked to re-authenticate during normal daily use.** This is a testable requirement, not a preference.
- `auth_session_expired` is an instrumented event (build spec §4.4) because it directly predicts a broken habit.
- Both parents may share a tablet: the home screen shows **whose account is active** in the top bar, with a one-tap switch. Account switching costs a click only when switching, never on the common path — the ≤3-click target holds for the logged-in user.

## 12. Accessibility

Non-negotiable, applied to every component:

- Body text ≥ 18pt; the current reading rendered very large
- Touch targets ≥ 44×44pt, primary actions larger
- WCAG 2.1 AA contrast minimum; test at 200% browser zoom
- Full keyboard operability; visible focus ring, never suppressed
- Status conveyed by **colour + text + icon**, never colour alone
- Respects `prefers-reduced-motion`
- `tabular-nums` on all readings so digits don't shift
- Numeric inputs use `inputmode="numeric"`, no spinners, no auto-advance between fields
