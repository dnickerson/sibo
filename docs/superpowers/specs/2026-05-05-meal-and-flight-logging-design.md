# Meal & Flight Logging — Design

## Goal

Add two new observational data points to the SIBO Protocol Tracker so the user can correlate meals and GA piloting events with symptoms (especially soft stools, hypothesized to correlate with cortisol from flying).

1. **Meal text field** for each of the three daily meals (Breakfast, Lunch, Dinner) — describing what was eaten.
2. **Flight log** for general aviation flights (the user is a GA pilot and sometimes flies twice a day).

Both must be timestamped and included in the JSON export for downstream evaluation.

## Data Model

All data continues to live in `localStorage` under `STORAGE_KEY = 'sibo_tracker_v2'`, keyed by local date string (`YYYY-MM-DD`).

Per-day record gains two fields:

```js
data["2026-05-05"] = {
  checks:     { ... },           // existing
  symptoms:   { ... },           // existing
  symptomLog: { ... },           // existing
  bristolLog: [ ... ],           // existing
  meals: {
    breakfast: { text: "scrambled eggs, sourdough, coffee", at: "2026-05-05T08:14:22.000Z" },
    lunch:     { text: "...",                                at: "..." },
    dinner:    { text: "...",                                at: "..." }
  },
  flightLog: [
    { at: "2026-05-05T09:14:00.000Z" },
    { at: "2026-05-05T15:22:00.000Z" }
  ]
}
```

- Empty meals are absent from `meals` (not stored as empty strings).
- `meals` keys (`breakfast`, `lunch`, `dinner`) match the existing `SCHEDULE` block ids.
- `flightLog` mirrors the existing `bristolLog` pattern: array of timestamped entries, newest appended.
- Future flight metadata (duration, conditions, etc.) can be added to entry objects without breaking existing records.

## UI — Today Timeline (Meals)

Each of the three meal `time-block`s (Breakfast / Lunch / Dinner) gets a Food block appended after its supplement items:

```
┌─ Breakfast ──────────── 8:00 AM ─┐
│  [✓] Vitamin D3 5,000 IU         │
│  [ ] Centrum Multivitamin        │
│  [ ] Ginger Tablet               │
│  [ ] Papaya Enzyme               │
│                                  │
│  ┌─ FOOD ─────────────────────┐  │
│  │ scrambled eggs, sourdough, │  │
│  │ coffee_                    │  │
│  └────────────────────────────┘  │
│  Logged 8:14 AM                  │
└──────────────────────────────────┘
```

**Visual treatment:**
- Small "FOOD" label above textarea — uppercase, letter-spaced, matches existing `.symptom-card-title` style.
- 2-row textarea with `auto-grow` (height adjusts as user types). Reuses existing `.notes-input` styling (surface2 background, accent3 focus border).
- Placeholder: `"What you ate (tap to dictate)"`.
- "Logged HH:MM AM" line below field, shown only after first save. Reuses `.recorded-at` styling.
- No checkbox (food is not a "complete" action — it's a record).
- No "Save" button — saves on blur (see Interactions).

## UI — Symptoms Tab (Flight Log)

A new card added at the top of the Symptoms tab, before "Daily Symptom Log":

```
┌─ FLIGHT LOG ────────────────┐
│  [ + Log Flight ]           │
│                             │
│  Flight    9:14 AM    ×     │
│  Flight    3:22 PM    ×     │
└─────────────────────────────┘
```

- "FLIGHT LOG" title — same `.symptom-card-title` style as other cards.
- "+ Log Flight" button — reuses existing `.log-entry-btn` style.
- Each entry shows "Flight" label, formatted local time, and a `×` remove button — reuses `.bristol-entry-item` markup pattern.
- Entries display newest-first.
- No form/fields on the button; one tap logs current `new Date().toISOString()`.

## Interactions

### Meal fields
- **Tap textarea** → device keyboard opens with mic for dictation.
- **Auto-save on blur** (`blur` event on textarea):
  - If text differs from last saved value: write `{ text, at: now }` to `data[key].meals[mealId]`, update displayed timestamp, show toast `"✓ Breakfast saved"` (capitalized meal name).
  - If text unchanged: no save, no toast.
- **Empty save**: blurring an empty field that was never saved is a no-op. Blurring after clearing a previously-saved field deletes the meal entry from `data[key].meals` (so the empty state is "absent" not "empty string").
- **On page load (timeline render)**: existing `meals[mealId].text` pre-fills the textarea; existing `meals[mealId].at` shows under the field as "Logged HH:MM AM".

### Flight log
- **Tap "+ Log Flight"** → appends `{ at: new Date().toISOString() }` to `data[key].flightLog`, re-renders list, shows toast `"✓ Flight logged"`.
- **Tap `×` on entry** → removes that entry by index, re-renders list, shows toast `"Entry removed"`. Matches existing Bristol remove flow.

### Reset Day
- Existing behavior **unchanged**: clears only `data[key].checks`. Meals and flight log persist (food and flights are observational history, not protocol intent).

## History Display

Each day's `.history-entry` card gains two optional sub-sections, rendered only when data is present.

**Meals** (after Bristol entries, before notes):
```
MEALS
Breakfast 8:14 AM · scrambled eggs, sourdough, coffee
Lunch     1:22 PM · grilled chicken, rice, salad
Dinner    6:45 PM · salmon, roasted veg
```
- Section header styled like the existing Bristol `.history-note` label ("MEALS · ").
- Each line: meal name, time, food text. Only renders for meals that have non-empty `text`. Section omitted entirely if no meals have text for the day.
- Long food text wraps; no truncation.

**Flights** (one line, after meals):
```
✈ 2 flights · 9:14 AM, 3:22 PM
```
- Inline single line in the day's card.
- Hidden entirely if `flightLog` is empty for that day.

## JSON Export

`exportData()` already serializes the entire localStorage object — no changes needed. New fields (`meals`, `flightLog`) appear automatically in the exported JSON file alongside existing fields.

## Implementation Notes

- All new code lives in `index.html` (the entire app is a single file — preserve that pattern).
- New CSS additions piggyback on existing variables (`--surface`, `--surface2`, `--border`, `--accent3`, etc.) for visual consistency.
- New JS functions follow existing naming conventions:
  - `saveMeal(mealId, text)`, `renderMealField(mealId)` (called from `renderTimeline`)
  - `logFlight()`, `renderFlightLog()`, `removeFlightEntry(idx)`
- Restoration on page load (analogous to existing `restoreSymptoms()` IIFE):
  - Meal textareas pre-fill from `data[key].meals` during `renderTimeline`.
  - Flight log renders from `data[key].flightLog` during initial Symptoms-tab render (called in init).

## Out of Scope (YAGNI)

- Dedicated food-only History view for cross-day pattern hunting (revisit when sufficient data accumulates).
- Multiple meal entries per meal slot (single editable entry per meal is sufficient).
- Flight metadata (duration, conditions, IFR/VFR, solo/dual). Schema leaves room to add later.
- Additional day flags (sleep, stress) beyond flying. Flight log structure is a pattern that can be cloned if needed.
- Server-side storage / sync. App remains localStorage-only.
