# Pulse — Design System

A calm, clinical, **accessibility-first** design system for a web-based blood pressure monitoring app built for elderly parents (primary user: low vision, slow movement, low tech comfort). 

> "Let's check in, Dad." — the tone we're after.

> **Status — revised 20 Sep 2026.** Scope, thresholds and storage were rewritten to match the current PRD and build spec (`BP Web App/docs/PULSE_PRD.md`, `PULSE_BUILD_SPEC.md`, `PULSE_LLD.md`). The visual foundations, voice and component anatomy below are unchanged and remain the reference. Where this document and the build spec disagree, **the build spec wins** — tell me and I'll fix it here.

## Product context

**Pulse** is a blood pressure web app for one family: two parents in New Zealand who log their own readings, and two family members (one in the UK, one in NZ) who can view them and are emailed when a reading is high.

### Screens

| # | Screen | State |
|---|---|---|
| 1 | **Home** (patient) — most recent reading, prominent, colour-coded. One glance answers "am I okay?" | Designed |
| 2 | **Log a reading** — a giant, forgiving number input. Date, time slot (AM/PM), systolic, diastolic, pulse. ≤ 3 clicks to save | Designed |
| 3 | **Trends** — line chart of systolic / diastolic / pulse over 7d / 30d / all | Designed |
| 4 | **Sign in** — one email field, one button. Magic link, no password | ⚠️ Undesigned |
| 5 | **Family dashboard** — both parents, latest reading and status each | ⚠️ Undesigned |
| 6 | **Patient detail** (family view) — one parent's chart and history | ⚠️ Undesigned |
| 7 | **Sharing** (patient) — who can see my readings; revoke access; alerts on/off | ⚠️ Undesigned |
| 8 | **Alert email** — the message sent when a reading is high | ⚠️ Undesigned |
| 9 | **GP report** — date range → PDF with chart and summary stats | ⚠️ Undesigned |
| 10 | **Health / delivery status** (family) — days logged, last alert, delivery failures | ⚠️ Undesigned |

Screens 4–10 are why this document can't yet be built from end to end. Screen 8 is the most important one in the product and currently has no design at all.

### Status bands

Colour-coded per reading. **Systolic only** — the GP has advised that diastolic is not a clinical concern for these patients, so diastolic is displayed prominently but never drives colour or an alert.

| Band | Condition | Colour |
|---|---|---|
| Normal | systolic ≤ 130 | healthy green |
| Elevated | systolic 131–150 | caution amber |
| **High** | **systolic > 150** | alert red |

Evaluated high → elevated → normal; first match wins.

**Red on screen and an email being sent are the same condition.** They must never disagree — a reading shown in red with no alert, or an alert for something shown in amber, destroys trust in both signals.

Out of scope for v1: mobile apps, device integration, medication reminders, voice input.

This system is a **personal project** — there is no existing codebase, Figma file, or prior brand. The design direction was chosen fresh based on:

- **Audience:** Elderly parents. 26px body minimum. 120px+ for the reading itself. Touch targets ≥ 64px. (The PRD states its floors in points — 18pt body, 44×44pt targets, i.e. ~24px and ~59px. These numbers are stricter, and they are the ones to build to. **Use px throughout; the pt figures are the PRD's floor, not a second standard.**)
- **Aesthetic:** Calm clinical — soft blues/greens, clean sans, generous whitespace, medical-grade trust.
- **Tone:** Warm and familial — first-name, second-person, reassuring.
- **Variation:** Two contrasting directions explored (see below).

### Two directions

- **Direction A — "Clinic":** Whites, soft blue-grays, crisp sans, low chroma. Feels like a well-designed waiting room. Primary choice for accessibility and trust.
- **Direction B — "Garden":** Bone paper, sage greens, warm terracotta accent. Feels like a journal on the kitchen table. Optional warmer variant.

Both share the same structure, type scale, spacing, and component anatomy — they differ only in palette and surface treatment.

## Sources & inspiration

No external design system was provided. Influences (not copied, not referenced assets):

- Apple Health (readings hierarchy)
- Calm, Oura (warm clinical restraint)
- NHS.uk frontend (elderly-friendly type + button sizing)
- Medical chart / patient portal conventions (systolic-over-diastolic reading format)

## Index

> ⚠️ **The files listed below are missing.** They are not in the vault, and the repo's `pulse-design-system/` contains only this document. The tokens, logo, icons and UI kit described here have to be rebuilt or located before anything can consume them. Until then, this document is a written specification, not a usable kit.

- **This file** — product context, content fundamentals, visual foundations, iconography.
- **colors_and_type.css** — CSS custom properties for both directions (A: Clinic, B: Garden) + semantic type classes. *Missing.*
- **assets/logo/** — Pulse wordmark + mark (SVG, uses currentColor). *Missing.*
- **assets/icons/** — 15 curated Lucide SVGs (1.5px stroke). *Missing — and Lucide can be loaded from its own package instead.*
- **preview/** — design-system cards (type specimens, palettes, components). *Missing.*

### Superseded prototype

An earlier UI kit (`ui_kits/web_app/`) demonstrated a home + log-reading click-through backed by **localStorage**, with CSV export and seeded Dec 2025–Mar 2026 data. It is **superseded** and should not be built on:

- The app now uses Supabase Postgres with per-user authentication; browser-local storage can't support two patients, two viewers or alerting.
- Production starts **empty** — no seeded historical readings.
- CSV export is not in the current MVP; the export requirement is a **PDF for the GP**.

Its component anatomy is still a good reference for structure: `ReadingCard`, `RangeTabs`, `TrendChart`, `ReadingList`, `NumberField`, `SlotToggle`, `StatusPill`, `TopBar`, `Button`, `Icon`.

## Content fundamentals

**Voice.** Warm and familial, like an adult child who is a nurse. We are speaking **to** the parent, never **about** them. Use their first name whenever we have it.

- ✅ "Let's check in, Dad."
- ✅ "Your morning reading is ready."
- ✅ "That's a little high today — how are you feeling?"
- ❌ "Patient BP elevated above threshold."
- ❌ "Click here to continue."
- ❌ "You are required to log readings daily."

**Person.** First-person plural for invitations ("Let's…"), second-person for instructions ("Tap the big button"). Avoid passive voice.

**Casing.** Sentence case everywhere — headings, buttons, labels. **Never** ALL CAPS except on a single letter badge (e.g. the "A" in avatar). Never title case on button labels.

**Numbers.** Spell out zero through nine in prose ("nine days in a row"). Use numerals in readings, always ("128 / 82"). Pulse uses a slash between systolic and diastolic with spaces around it. Units are always lowercase ("mmhg" in small print, "bpm" for heart rate — in small print only; the big number has no unit).

**Length.** Short. A home-screen label is 1–5 words. A button is 2–4 words. A confirmation message is one sentence.

**Emoji.** Sparingly and warmly. ❤️ for a healthy streak, 👍 for a successful log, 🌤️ for "good morning." Never as bullet points, never as status indicators (use color + icon for those). One emoji per view, maximum.

**Accessibility language.** Never say "elderly" to the user. Never say "senior." We say "you" or their name. Don't explain _why_ the buttons are big.

### Copy examples

|Context|Copy|
|---|---|
|Home greeting, morning|"Good morning, Dad."|
|Home greeting, returning|"Welcome back."|
|Log reading CTA|"Log a reading"|
|Input placeholder (systolic)|"Top number"|
|Input placeholder (diastolic)|"Bottom number"|
|Save button|"Save reading"|
|Success toast|"Saved. Nice job. 👍"|
|Out of range (high)|"A little high today. How are you feeling?"|
|Out of range (low)|"A little low today. Sit for a moment before standing."|
|Empty state|"No readings yet. Let's add the first one."|
|Streak|"That's 7 days in a row. ❤️"|

### Copy still to write

Drafts for the undesigned screens, in voice — treat as starting points, not decisions:

|Context|Draft copy|
|---|---|
|Sign-in prompt|"Pop in your email and we'll send you a link."|
|Sign-in sent|"Check your email — the link's on its way."|
|Replace an existing reading|"You already logged a morning reading today — 142 / 88. Replace it?"|
|Save failed|"That didn't save. Your numbers are still here — try again?"|
|Family dashboard, all well|"Both looking steady."|
|Family dashboard, high reading|"Dad's morning reading was high."|
|Alert email subject|"Pulse: Dad's reading is 168 / 95 (Tue 8:05am NZDT)"|
|Sharing screen|"People who can see your readings"|
|Revoke confirmation|"Stop sharing with Sarah? She won't see new readings or get alerts."|
|Alert delivery failed|"We couldn't deliver the last alert email. Check the address?"|

**Two voice rules for the family-facing screens.** The warm, first-name voice is for the *parent*. Alerts and the dashboard are read by an adult child who may be at work and needs the facts fast — plain, calm, specific, no emoji, no reassurance the data doesn't support. Never soften a high reading in an alert.

## Visual foundations

### Color

Two palettes — both low-chroma, high-contrast at the type level, with a single warm accent reserved for positive moments.

**Direction A — Clinic** (default)

- Surfaces: #FBFCFD page, #FFFFFF cards, #EEF3F7 subtle panel
- Ink: #0F1B2D primary text, #3B4A5E secondary text, #6B7A8F tertiary
- Brand blue: #2B6CB0 (links, primary buttons)
- Healthy green: #3F8A5C (in-range reading)
- Caution amber: #C08A2E (mildly elevated)
- Alert red: #B04A3A (high — never bright, never urgent-looking)

**Direction B — Garden** (warm alternate)

- Surfaces: #F6F1E8 page, #FBF7EE cards, #EFE7D6 subtle panel
- Ink: #2A2823 primary text, #55524A secondary
- Sage: #6B7F5C (primary)
- Terracotta: #C47A5A (accent, used sparingly)
- Healthy, caution, alert: same semantic hues as Clinic but slightly warmer.

All colors pass **WCAG AAA for large text** against their paired surface. Body text passes AA minimum.

### Typography

- **Display + body:** **Inter Tight** (sans, high x-height, excellent at very large sizes). 500 weight for headings and readings, 400 for body.
- **Numerals:** font-variant-numeric: tabular-nums for all readings and timestamps — numbers never shift as they change.

Type scale (accessibility-max):

|Role|Size|Weight|Line-height|
|---|---|---|---|
|Reading hero|144px|500|1.0|
|Reading secondary|72px|500|1.0|
|H1|48px|500|1.15|
|H2|36px|500|1.2|
|H3|28px|500|1.3|
|Body|26px|400|1.5|
|Small|22px|400|1.45|
|Micro (units only)|18px|500|1.4|

No text is ever below 22px in the UI except for unit labels ("mmhg", "bpm") that sit next to a much larger number.

### Spacing

Base unit: **8px**. Scale: 4, 8, 12, 16, 24, 32, 48, 64, 96, 128. Most card paddings are 32 or 48. Stack spacing between unrelated sections is 64 or 96.

### Backgrounds

Flat. No gradients, no photography, no patterns, no textures. The page is a single soft tint; cards are slightly lighter. That's it. Trust comes from emptiness.

### Corners and borders

- Card radius: **20px** — generous, friendly, never pill-shaped.
- Button radius: **16px**.
- Input radius: **16px**.
- Borders: 1.5px solid on the --line token. Borders are preferred over shadows for separating surfaces.

### Shadows

Minimal. One elevation token (--shadow-card) — a very soft, vertical-only shadow (0 2px 8px rgba(15, 27, 45, 0.04)). Used on the main reading card and nowhere else. No hover-lift shadows.

### Hover / press / focus

- **Hover (buttons):** surface darkens ~6%. No scale, no shadow.
- **Press:** surface darkens ~12% and the button translates down 1px. No scale-down — elderly users can mis-trigger.
- **Focus:** a **thick 3px outline** in brand blue, 3px offset. Always visible on keyboard focus. Never suppressed.
- **Disabled:** 50% opacity, cursor default. No aria-disabled without aria-label explaining why.

### Animation

Sparse. Only two motion tokens:

- **--ease-calm:** cubic-bezier(0.4, 0, 0.2, 1) — for layout changes.
- **Duration:** 240ms for entrances, 160ms for exits. Never longer. Never bouncy.

Toasts fade + rise 8px. Buttons do not animate. Readings appear with no animation — they must feel **printed**, not animated.

prefers-reduced-motion disables everything except opacity transitions.

### Layout rules

- Max content width: **720px** centered. On wider screens, everything stays centered and generously padded.
- Single-column always. Multi-column reads as a form, and forms feel clinical in a bad way.
- One primary action per screen. Always at the bottom, always full-width on small screens, always ≥ 72px tall.
- Fixed elements: a minimal top bar with the Pulse mark and the day. No bottom nav.
- Touch targets: **≥ 64px** every time. Button padding is 20px vertical minimum.

### Transparency and blur

Never used. Elderly users are helped by **clear, opaque surfaces**. No glass, no frost, no scrim.

### Cards

- White (or bone) fill
- 1.5px border in --line
- 20px radius
- 32px or 48px padding
- One elevation max (the reading card only)

## Iconography

**Lucide** (https://lucide.dev) — clean, modern, 1.5px stroke. Matches the quiet clinical tone without being cold.

Loading: via CDN reference in HTML kits (https://unpkg.com/lucide@latest). A small curated subset is saved as SVGs in assets/icons/ for offline artifacts (slides, PDFs, screenshots).

Curated subset for this product:

- heart — brand mark accent, streaks
- activity — pulse/trend
- plus — log a reading
- check — success
- alert-triangle — out-of-range warning (never as panic)
- arrow-right, arrow-left — navigation
- calendar — history (future)
- user — profile (future)
- sun / moon — morning / evening greetings

**Stroke weight:** 1.5px. Always. Never mix in thinner or filled icons.

**Icon size:** 28px in buttons, 32px inline with body text, 64px in empty states. Icons scale with adjacent type — never free-floating.

**Emoji:** allowed warmly for human moments only (❤️, 👍, 🌤️). Never for status (use color + Lucide icon).

**Unicode chars:** the slash / between systolic and diastolic is the real character / set in tabular-nums. No fancy division slash.

### Substitutions / flags

- ⚠️ **Fonts:** No custom fonts were provided. Using **Inter Tight** from Google Fonts (closest match to the clean clinical direction). If you want something more distinctive (e.g. a softer humanist sans), drop a .woff2 into fonts/ and update --font-sans in colors_and_type.css.
- ⚠️ **Logo:** A simple wordmark + heart glyph is included in assets/logo/. Hand-drawn; trivial to swap.
- ⚠️ **Illustrations:** None included. Empty states use type + a single Lucide icon. If you want a warm spot illustration on the home screen, commission or license one.

## For maintainers

If adding a new screen:

1. Start from an existing kit component.
2. Body text ≥ 26px. Touch targets ≥ 64px.
3. One primary action per screen.
4. Read the copy aloud — if it sounds like a hospital form, rewrite it.
5. Test with screen reader and keyboard only.
