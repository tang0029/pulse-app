# Pulse â€” Design System

A calm, clinical, **accessibility-first** design system for a web-based blood pressure monitoring app built for elderly parents (primary user: low vision, shaky hands, low tech comfort). A secondary caregiver view is out of scope for v1.

> "Let's check in, Dad." â€” the tone we're after.

## Product context

**Pulse** is a browser-based blood pressure journal. No account, no install, no backend â€” readings live in localStorage. From the product brief (Apr 2026, Tiara):

1. **Home** â€” shows the most recent reading prominently, plus a line chart of systolic/diastolic trends over 7d / 30d / all. One glance answers "am I okay?"
2. **Log a reading** â€” a giant, forgiving number input. Date, time slot (AM/PM), systolic, diastolic, pulse. â‰¤ 3 clicks to save.
3. **Export** â€” CSV download of all readings, because localStorage is fragile.

Color-coded status per reading: **normal** (healthy green), **elevated** (caution amber), **high** â€” systolic â‰¥ 160 is clearly flagged in alert red.

Out of scope for v1: mobile apps, accounts, cloud sync, caregiver sharing, device integration, medication reminders.

This system is a **personal project** â€” there is no existing codebase, Figma file, or prior brand. The design direction was chosen fresh based on:

- **Audience:** Elderly parents. 26px body minimum. 120px+ for the reading itself. Touch targets â‰¥ 64px.
- **Aesthetic:** Calm clinical â€” soft blues/greens, clean sans, generous whitespace, medical-grade trust.
- **Tone:** Warm and familial â€” first-name, second-person, reassuring.
- **Variation:** Two contrasting directions explored (see below).

### Two directions

- **Direction A â€” "Clinic":** Whites, soft blue-grays, crisp sans, low chroma. Feels like a well-designed waiting room. Primary choice for accessibility and trust.
- **Direction B â€” "Garden":** Bone paper, sage greens, warm terracotta accent. Feels like a journal on the kitchen table. Optional warmer variant.

Both share the same structure, type scale, spacing, and component anatomy â€” they differ only in palette and surface treatment.

## Sources & inspiration

No external design system was provided. Influences (not copied, not referenced assets):

- Apple Health (readings hierarchy)
- Calm, Oura (warm clinical restraint)
- NHS.uk frontend (elderly-friendly type + button sizing)
- Medical chart / patient portal conventions (systolic-over-diastolic reading format)

## Index

Everything lives at the project root unless noted:

- **README.md** â€” this file. Product context, content fundamentals, visual foundations, iconography.
- **colors_and_type.css** â€” CSS custom properties for both directions (A: Clinic, B: Garden) + semantic type classes.
- **SKILL.md** â€” cross-compatible skill file so this can be used as an Agent Skill.
- **assets/logo/** â€” Pulse wordmark + mark (SVG, uses currentColor).
- **assets/icons/** â€” 15 curated Lucide SVGs (1.5px stroke).
- **preview/** â€” design-system cards (type specimens, palettes, components, etc). Rendered in the Design System tab.
- **ui_kits/web_app/** â€” the main Pulse web app UI kit (home + log-reading click-through with localStorage + CSV export + seeded Dec 2025â€“Mar 2026 data).

### Preview cards

Type, Colors, Spacing, Components, Brand â€” split into small single-purpose cards visible in the Design System tab.

### UI kit â€” web app

- index.html â€” entry
- data.js â€” seed data + classify/format helpers
- Components.jsx â€” Icon, TopBar, Button, StatusPill
- HomeComponents.jsx â€” ReadingCard, RangeTabs, TrendChart, ReadingList
- LogForm.jsx â€” NumberField, SlotToggle, LogForm
- App.jsx â€” wiring, view routing, direction toggle

## Content fundamentals

**Voice.** Warm and familial, like an adult child who is a nurse. We are speaking **to** the parent, never **about** them. Use their first name whenever we have it.

- âœ… "Let's check in, Dad."
- âœ… "Your morning reading is ready."
- âœ… "That's a little high today â€” how are you feeling?"
- âŒ "Patient BP elevated above threshold."
- âŒ "Click here to continue."
- âŒ "You are required to log readings daily."

**Person.** First-person plural for invitations ("Let'sâ€¦"), second-person for instructions ("Tap the big button"). Avoid passive voice.

**Casing.** Sentence case everywhere â€” headings, buttons, labels. **Never** ALL CAPS except on a single letter badge (e.g. the "A" in avatar). Never title case on button labels.

**Numbers.** Spell out zero through nine in prose ("nine days in a row"). Use numerals in readings, always ("128 / 82"). Pulse uses a slash between systolic and diastolic with spaces around it. Units are always lowercase ("mmhg" in small print, "bpm" for heart rate â€” in small print only; the big number has no unit).

**Length.** Short. A home-screen label is 1â€“5 words. A button is 2â€“4 words. A confirmation message is one sentence.

**Emoji.** Sparingly and warmly. â¤ï¸ for a healthy streak, ðŸ‘ for a successful log, ðŸŒ¤ï¸ for "good morning." Never as bullet points, never as status indicators (use color + icon for those). One emoji per view, maximum.

**Accessibility language.** Never say "elderly" to the user. Never say "senior." We say "you" or their name. Don't explain *why* the buttons are big.

### Copy examples

| Context | Copy |
| --- | --- |
| Home greeting, morning | "Good morning, Dad." |
| Home greeting, returning | "Welcome back." |
| Log reading CTA | "Log a reading" |
| Input placeholder (systolic) | "Top number" |
| Input placeholder (diastolic) | "Bottom number" |
| Save button | "Save reading" |
| Success toast | "Saved. Nice job. ðŸ‘" |
| Out of range (high) | "A little high today. How are you feeling?" |
| Out of range (low) | "A little low today. Sit for a moment before standing." |
| Empty state | "No readings yet. Let's add the first one." |
| Streak | "That's 7 days in a row. â¤ï¸" |

## Visual foundations

### Color

Two palettes â€” both low-chroma, high-contrast at the type level, with a single warm accent reserved for positive moments.

**Direction A â€” Clinic** (default)

- Surfaces: #FBFCFD page, #FFFFFF cards, #EEF3F7 subtle panel
- Ink: #0F1B2D primary text, #3B4A5E secondary text, #6B7A8F tertiary
- Brand blue: #2B6CB0 (links, primary buttons)
- Healthy green: #3F8A5C (in-range reading)
- Caution amber: #C08A2E (mildly elevated)
- Alert red: #B04A3A (high â€” never bright, never urgent-looking)

**Direction B â€” Garden** (warm alternate)

- Surfaces: #F6F1E8 page, #FBF7EE cards, #EFE7D6 subtle panel
- Ink: #2A2823 primary text, #55524A secondary
- Sage: #6B7F5C (primary)
- Terracotta: #C47A5A (accent, used sparingly)
- Healthy, caution, alert: same semantic hues as Clinic but slightly warmer.

All colors pass **WCAG AAA for large text** against their paired surface. Body text passes AA minimum.

### Typography

- **Display + body:** **Inter Tight** (sans, high x-height, excellent at very large sizes). 500 weight for headings and readings, 400 for body.
- **Numerals:** font-variant-numeric: tabular-nums for all readings and timestamps â€” numbers never shift as they change.

Type scale (accessibility-max):

| Role | Size | Weight | Line-height |
| --- | --- | --- | --- |
| Reading hero | 144px | 500 | 1.0 |
| Reading secondary | 72px | 500 | 1.0 |
| H1 | 48px | 500 | 1.15 |
| H2 | 36px | 500 | 1.2 |
| H3 | 28px | 500 | 1.3 |
| Body | 26px | 400 | 1.5 |
| Small | 22px | 400 | 1.45 |
| Micro (units only) | 18px | 500 | 1.4 |

No text is ever below 22px in the UI except for unit labels ("mmhg", "bpm") that sit next to a much larger number.

### Spacing

Base unit: **8px**. Scale: 4, 8, 12, 16, 24, 32, 48, 64, 96, 128. Most card paddings are 32 or 48. Stack spacing between unrelated sections is 64 or 96.

### Backgrounds

Flat. No gradients, no photography, no patterns, no textures. The page is a single soft tint; cards are slightly lighter. That's it. Trust comes from emptiness.

### Corners and borders

- Card radius: **20px** â€” generous, friendly, never pill-shaped.
- Button radius: **16px**.
- Input radius: **16px**.
- Borders: 1.5px solid on the --line token. Borders are preferred over shadows for separating surfaces.

### Shadows

Minimal. One elevation token (--shadow-card) â€” a very soft, vertical-only shadow (0 2px 8px rgba(15, 27, 45, 0.04)). Used on the main reading card and nowhere else. No hover-lift shadows.

### Hover / press / focus

- **Hover (buttons):** surface darkens \~6%. No scale, no shadow.
- **Press:** surface darkens \~12% and the button translates down 1px. No scale-down â€” elderly users can mis-trigger.
- **Focus:** a **thick 3px outline** in brand blue, 3px offset. Always visible on keyboard focus. Never suppressed.
- **Disabled:** 50% opacity, cursor default. No aria-disabled without aria-label explaining why.

### Animation

Sparse. Only two motion tokens:

- **--ease-calm:** cubic-bezier(0.4, 0, 0.2, 1) â€” for layout changes.
- **Duration:** 240ms for entrances, 160ms for exits. Never longer. Never bouncy.

Toasts fade + rise 8px. Buttons do not animate. Readings appear with no animation â€” they must feel **printed**, not animated.

prefers-reduced-motion disables everything except opacity transitions.

### Layout rules

- Max content width: **720px** centered. On wider screens, everything stays centered and generously padded.
- Single-column always. Multi-column reads as a form, and forms feel clinical in a bad way.
- One primary action per screen. Always at the bottom, always full-width on small screens, always â‰¥ 72px tall.
- Fixed elements: a minimal top bar with the Pulse mark and the day. No bottom nav.
- Touch targets: **â‰¥ 64px** every time. Button padding is 20px vertical minimum.

### Transparency and blur

Never used. Elderly users are helped by **clear, opaque surfaces**. No glass, no frost, no scrim.

### Cards

- White (or bone) fill
- 1.5px border in --line
- 20px radius
- 32px or 48px padding
- One elevation max (the reading card only)

## Iconography

**Lucide** (https://lucide.dev) â€” clean, modern, 1.5px stroke. Matches the quiet clinical tone without being cold.

Loading: via CDN reference in HTML kits (https://unpkg.com/lucide@latest). A small curated subset is saved as SVGs in assets/icons/ for offline artifacts (slides, PDFs, screenshots).

Curated subset for this product:

- heart â€” brand mark accent, streaks
- activity â€” pulse/trend
- plus â€” log a reading
- check â€” success
- alert-triangle â€” out-of-range warning (never as panic)
- arrow-right, arrow-left â€” navigation
- calendar â€” history (future)
- user â€” profile (future)
- sun / moon â€” morning / evening greetings

**Stroke weight:** 1.5px. Always. Never mix in thinner or filled icons.

**Icon size:** 28px in buttons, 32px inline with body text, 64px in empty states. Icons scale with adjacent type â€” never free-floating.

**Emoji:** allowed warmly for human moments only (â¤ï¸, ðŸ‘, ðŸŒ¤ï¸). Never for status (use color + Lucide icon).

**Unicode chars:** the slash / between systolic and diastolic is the real character / set in tabular-nums. No fancy division slash.

### Substitutions / flags

- âš ï¸ **Fonts:** No custom fonts were provided. Using **Inter Tight** from Google Fonts (closest match to the clean clinical direction). If you want something more distinctive (e.g. a softer humanist sans), drop a .woff2 into fonts/ and update --font-sans in colors_and_type.css.
- âš ï¸ **Logo:** A simple wordmark + heart glyph is included in assets/logo/. Hand-drawn; trivial to swap.
- âš ï¸ **Illustrations:** None included. Empty states use type + a single Lucide icon. If you want a warm spot illustration on the home screen, commission or license one.

## For maintainers

If adding a new screen:

1. Start from an existing kit component.
2. Body text â‰¥ 26px. Touch targets â‰¥ 64px.
3. One primary action per screen.
4. Read the copy aloud â€” if it sounds like a hospital form, rewrite it.
5. Test with screen reader and keyboard only.
