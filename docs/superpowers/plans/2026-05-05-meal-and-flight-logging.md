# Meal & Flight Logging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-meal food text fields (Breakfast/Lunch/Dinner) on the Today timeline, and a flight log on the Symptoms tab, both feeding into the existing JSON export.

**Architecture:** Extend the existing single-file PWA at `index.html`. New data lands in the per-day record under two new keys: `meals` (object keyed by mealId) and `flightLog` (array of `{at}` entries, mirroring `bristolLog`). UI follows existing patterns (`.notes-input`, `.symptom-card`, `.bristol-entry-item`, `.log-entry-btn`). No new files, no new dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS, localStorage (`STORAGE_KEY = 'sibo_tracker_v2'`), single-file PWA. No build step. No test framework. Verification is manual in-browser plus localStorage inspection in DevTools.

**Reference spec:** `docs/superpowers/specs/2026-05-05-meal-and-flight-logging-design.md`

---

## File Structure

All changes are in **one file**: `/home/dananickerson/sibo-pwa/index.html`.

Changes group into four logical regions of that file:

1. **`<style>` block** (lines ~17–755): new CSS rules for `.food-block`, `.food-label`, `.food-input`, `.food-recorded-at`, `.flight-entry`.
2. **`<script>` data + state region** (lines ~941–1085): no structural change; new helpers (`saveMeal`, `getMeal`, `logFlight`, `renderFlightLog`, `removeFlightEntry`) added in their own section.
3. **`renderTimeline()`** (lines ~1108–1147): inject a food block into meal time-blocks.
4. **`renderHistory()`** (lines ~1320–1380): append meals and flight lines per day.
5. **HTML body** (lines ~793–887): insert the Flight Log card at the top of the Symptoms tab section.

Version string bumps `v1.4` → `v1.5` once.

---

## Pre-Implementation Setup

### Task 0: Verify clean working tree, prepare local test server

**Files:** none

- [ ] **Step 1: Confirm clean working tree on `main`**

```bash
cd /home/dananickerson/sibo-pwa
git status
```

Expected: `nothing to commit, working tree clean`. The most recent commit should be the spec commit (`spec: meal and flight logging design`). If the tree is dirty, stop and resolve before proceeding.

- [ ] **Step 2: Start a local static server in the project root**

The PWA includes a service worker (`sw.js`) which only registers when served via HTTP, not from a `file://` URL. Use Python's built-in server in the background:

```bash
cd /home/dananickerson/sibo-pwa
python3 -m http.server 8000
```

Run this in a background terminal (or via `run_in_background`). Leave it running for the duration of implementation. URL: `http://localhost:8000/`.

- [ ] **Step 3: Open the app, confirm baseline works**

Open `http://localhost:8000/` in a browser. Verify:
- Header shows "SIBO Protocol" with current time and date.
- Today timeline renders with all SCHEDULE blocks (Pre-Breakfast, Breakfast, Pre-Lunch, …).
- Symptoms tab loads with the existing 5 scales + Bristol grid + Notes.
- History tab loads (may be empty).

If any of those fail, stop — the baseline is broken. Otherwise proceed.

- [ ] **Step 4: Open DevTools and inspect localStorage**

In DevTools → Application → Local Storage → `http://localhost:8000` → key `sibo_tracker_v2`. Note the existing shape (object keyed by `YYYY-MM-DD`). You'll inspect this throughout the plan.

---

## Task 1: Add CSS for food block and flight entry rows

**Files:**
- Modify: `index.html` — add CSS rules at the end of the `<style>` block (immediately before `</style>`)

- [ ] **Step 1: Add new CSS rules**

Locate the end of the `<style>` block (just before `</style>`, around line 755). Insert the following rules immediately before `</style>`:

```css
  /* Food block (per-meal, on Today timeline) */
  .food-block {
    margin: 6px 0 0;
    padding: 10px 12px;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 6px;
  }

  .food-label {
    font-size: 10px;
    letter-spacing: 2px;
    text-transform: uppercase;
    color: var(--text2);
    margin-bottom: 6px;
  }

  .food-input {
    width: 100%;
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: 4px;
    padding: 8px 10px;
    color: var(--text);
    font-family: 'DM Mono', monospace;
    font-size: 13px;
    line-height: 1.5;
    resize: none;
    transition: border-color 0.2s;
    min-height: 44px;
  }

  .food-input:focus {
    outline: none;
    border-color: var(--accent3);
  }

  .food-input::placeholder { color: var(--text3); }

  .food-recorded-at {
    font-size: 10px;
    color: var(--text3);
    margin-top: 4px;
    text-align: right;
    letter-spacing: 0.5px;
  }

  .food-recorded-at span { color: var(--accent2); }

  /* Flight log entry — reuses .bristol-entry-item, no new rules needed */

  /* History meals/flights sub-sections */
  .history-meals,
  .history-flights {
    border-top: 1px solid var(--border);
    padding-top: 8px;
    margin-top: 6px;
    font-size: 12px;
    color: var(--text2);
    line-height: 1.5;
  }

  .history-meals-label,
  .history-flights-label {
    color: var(--text3);
    font-size: 10px;
    letter-spacing: 1px;
    text-transform: uppercase;
    margin-right: 6px;
  }

  .history-meal-line {
    display: block;
  }

  .history-meal-name {
    color: var(--accent2);
    font-weight: 500;
  }

  .history-meal-time {
    color: var(--text3);
    font-size: 11px;
    margin: 0 6px;
  }
```

- [ ] **Step 2: Reload browser, verify no visual regression**

Hard reload (`Cmd/Ctrl+Shift+R`) `http://localhost:8000/`. Verify:
- Today timeline still renders correctly.
- Symptoms tab still renders correctly.
- No CSS errors in DevTools Console.

The new rules are not yet referenced by any markup, so the page should look unchanged.

- [ ] **Step 3: Commit**

```bash
cd /home/dananickerson/sibo-pwa
git add index.html
git commit -m "feat(css): add styles for food block and history meal/flight rows"
```

---

## Task 2: Add data-layer helpers for meals and flights

**Files:**
- Modify: `index.html` — add new helper functions in the `<script>` block

- [ ] **Step 1: Insert a new `// ─── MEALS ───` section after the `// ─── STATE ───` block**

Locate the `// ─── STATE ───` section (around line 1054). After the `setCheck` function (ends around line 1085, just before `// ─── CLOCK ───`), insert this block:

```javascript
// ─── MEALS ──────────────────────────────────────────────────────────────────

function getMeal(mealId) {
  const data = loadData();
  const key = getTodayKey();
  return data[key]?.meals?.[mealId] || null;
}

function saveMeal(mealId, text) {
  const data = loadData();
  const key = getTodayKey();
  if (!data[key]) data[key] = { checks: {}, symptoms: null };
  if (!data[key].meals) data[key].meals = {};
  const trimmed = text.trim();
  if (trimmed === '') {
    delete data[key].meals[mealId];
  } else {
    data[key].meals[mealId] = { text: trimmed, at: new Date().toISOString() };
  }
  saveData(data);
}

// ─── FLIGHT LOG ─────────────────────────────────────────────────────────────

function logFlight() {
  const data = loadData();
  const key = getTodayKey();
  if (!data[key]) data[key] = { checks: {}, symptoms: null };
  if (!data[key].flightLog) data[key].flightLog = [];
  data[key].flightLog.push({ at: new Date().toISOString() });
  saveData(data);
  renderFlightLog();
  showToast('✓ Flight logged');
}

function renderFlightLog() {
  const data = loadData();
  const key = getTodayKey();
  const log = data[key]?.flightLog || [];
  const container = document.getElementById('flight-log-list');
  if (!container) return;
  if (log.length === 0) { container.innerHTML = ''; return; }
  container.innerHTML = log.slice().reverse().map((entry, i) => `
    <div class="bristol-entry-item">
      <span class="bristol-entry-type">Flight</span>
      <span class="bristol-entry-time">${fmtTime(entry.at)}</span>
      <button onclick="removeFlightEntry(${log.length - 1 - i})" style="background:none;border:none;color:var(--text3);cursor:pointer;font-size:14px;padding:0 4px;" title="Remove">×</button>
    </div>`).join('');
}

function removeFlightEntry(idx) {
  const data = loadData();
  const key = getTodayKey();
  if (!data[key]?.flightLog) return;
  data[key].flightLog.splice(idx, 1);
  saveData(data);
  renderFlightLog();
  showToast('Entry removed');
}
```

- [ ] **Step 2: Reload browser, verify functions are defined**

Hard reload `http://localhost:8000/`. Open DevTools Console. Run:

```javascript
typeof saveMeal
typeof getMeal
typeof logFlight
typeof renderFlightLog
typeof removeFlightEntry
```

Expected: all five return `"function"`.

- [ ] **Step 3: Smoke-test the data layer in the console**

In the DevTools console:

```javascript
saveMeal('breakfast', 'test eggs');
getMeal('breakfast');
// Expected: { text: "test eggs", at: "2026-05-05T..." }

logFlight();
// Expected: toast appears (briefly), but no UI yet — that's fine.

JSON.parse(localStorage.sibo_tracker_v2);
// Expected: today's record contains `meals.breakfast` and `flightLog: [{at: ...}]`.
```

Then clean up the test data:

```javascript
const k = (() => { const n = new Date(); return `${n.getFullYear()}-${String(n.getMonth()+1).padStart(2,'0')}-${String(n.getDate()).padStart(2,'0')}`; })();
const d = JSON.parse(localStorage.sibo_tracker_v2);
delete d[k].meals;
delete d[k].flightLog;
localStorage.sibo_tracker_v2 = JSON.stringify(d);
```

Reload — confirm localStorage no longer contains test entries.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(data): add meal and flight-log persistence helpers"
```

---

## Task 3: Render food textarea in each meal time-block on Today

**Files:**
- Modify: `index.html` — `renderTimeline()` function (around lines 1108–1147)

- [ ] **Step 1: Update `renderTimeline()` to inject a food block for meal blocks**

Find the `renderTimeline()` function. Locate the section where `block.items.forEach(...)` runs (around line 1127). After the `forEach` loop and BEFORE the closing `html += '</div>';` (which closes the time-block div), insert the food-block injection.

Replace this section:

```javascript
    block.items.forEach(item => {
      const isChecked = !!checks[item.id];
      if (item.tag !== 'prn') { total++; if (isChecked) done++; }
      const tagLabels = { required: 'Active', discuss: 'Discuss w/ MD', prn: 'As Needed', behavior: 'Behavior' };
      html += `
        <div class="reminder-item ${item.type} ${isChecked ? 'completed' : ''}" onclick="toggleCheck('${item.id}', this)">
          <div class="check-box"><span class="check-icon">✓</span></div>
          <div class="reminder-content">
            <div class="reminder-name">${item.name}</div>
            <div class="reminder-detail">${item.detail}</div>
          </div>
          <div class="reminder-tag tag-${item.tag}">${tagLabels[item.tag]}</div>
        </div>`;
    });

    html += `</div>`;
  });
```

with:

```javascript
    block.items.forEach(item => {
      const isChecked = !!checks[item.id];
      if (item.tag !== 'prn') { total++; if (isChecked) done++; }
      const tagLabels = { required: 'Active', discuss: 'Discuss w/ MD', prn: 'As Needed', behavior: 'Behavior' };
      html += `
        <div class="reminder-item ${item.type} ${isChecked ? 'completed' : ''}" onclick="toggleCheck('${item.id}', this)">
          <div class="check-box"><span class="check-icon">✓</span></div>
          <div class="reminder-content">
            <div class="reminder-name">${item.name}</div>
            <div class="reminder-detail">${item.detail}</div>
          </div>
          <div class="reminder-tag tag-${item.tag}">${tagLabels[item.tag]}</div>
        </div>`;
    });

    if (block.id === 'breakfast' || block.id === 'lunch' || block.id === 'dinner') {
      const meal = getMeal(block.id);
      const text = meal?.text || '';
      const stamp = meal?.at ? fmtTime(meal.at) : '';
      const safeText = text.replace(/&/g, '&amp;').replace(/</g, '&lt;');
      html += `
        <div class="food-block">
          <div class="food-label">Food</div>
          <textarea
            class="food-input"
            id="food-${block.id}"
            rows="2"
            placeholder="What you ate (tap to dictate)"
            onblur="handleMealBlur('${block.id}', this)"
          >${safeText}</textarea>
          <div class="food-recorded-at" id="food-${block.id}-stamp" style="display:${stamp ? 'block' : 'none'}">Logged <span>${stamp}</span></div>
        </div>`;
    }

    html += `</div>`;
  });
```

- [ ] **Step 2: Add `handleMealBlur` function**

In the `// ─── MEALS ───` section (added in Task 2), append after `saveMeal`:

```javascript
function handleMealBlur(mealId, el) {
  const text = el.value;
  const prev = getMeal(mealId);
  const prevText = prev?.text || '';
  if (text.trim() === prevText.trim()) return; // unchanged — no save
  saveMeal(mealId, text);
  const stampEl = document.getElementById(`food-${mealId}-stamp`);
  const updated = getMeal(mealId);
  if (updated) {
    stampEl.querySelector('span').textContent = fmtTime(updated.at);
    stampEl.style.display = 'block';
  } else {
    stampEl.style.display = 'none';
  }
  const labelMap = { breakfast: 'Breakfast', lunch: 'Lunch', dinner: 'Dinner' };
  showToast(`✓ ${labelMap[mealId]} saved`);
}
```

- [ ] **Step 3: Reload and verify food block renders for the three meal time-blocks**

Hard reload the page. On the Today timeline, verify:
- The Breakfast block has a "Food" textarea after its supplements.
- The Lunch block has a "Food" textarea after its supplements.
- The Dinner block has a "Food" textarea after its supplements.
- The other time-blocks (Pre-Breakfast, Pre-Lunch, Post-Lunch, Pre-Dinner, Evening, Bedtime, As Needed) do **not** have a food textarea.

- [ ] **Step 4: Verify save-on-blur**

Type into the Breakfast textarea: `scrambled eggs, sourdough, coffee`. Tap or click outside (e.g., on the Pre-Breakfast block).

Verify:
- Toast appears: `✓ Breakfast saved`.
- "Logged HH:MM AM" appears below the textarea.
- DevTools → localStorage → `sibo_tracker_v2` → today's record contains `meals.breakfast: { text: "scrambled eggs, sourdough, coffee", at: "..." }`.

Hard reload. Verify the textarea pre-fills with the saved text and the timestamp is still shown.

- [ ] **Step 5: Verify no-change blur is a no-op**

Click into the Breakfast textarea (without changing anything), then click out. Verify:
- No toast.
- localStorage timestamp does NOT update (compare `at` value to step 4).

- [ ] **Step 6: Verify clearing the field deletes the meal entry**

Clear all text from the Breakfast textarea, then click out. Verify:
- Toast appears: `✓ Breakfast saved`.
- Timestamp line disappears.
- localStorage no longer has `meals.breakfast` (or `meals` is `{}`).

Note: the toast on clearing technically says "saved" but the action is "deleted." This is acceptable — the user just typed nothing. If this feels wrong during execution, swap the toast for a no-op when clearing.

- [ ] **Step 7: Verify XSS-safe rendering**

In the Breakfast textarea, type: `<img src=x>` and blur. Reload. Verify:
- The textarea displays the literal text `<img src=x>`.
- No image element appears in the DOM (right-click → inspect the textarea — it should contain text content, not a child `<img>` element).

Then clear the field for cleanliness.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(today): add per-meal food textarea with save-on-blur"
```

---

## Task 4: Add Flight Log card to Symptoms tab

**Files:**
- Modify: `index.html` — Symptoms tab section in HTML body (around lines 793–887)

- [ ] **Step 1: Insert the Flight Log card at the top of the Symptoms tab**

Locate the `<div class="section" id="section-symptoms">` block (around line 793). Inside it, find the opening `<div class="symptom-section">` and insert the new card as the FIRST `.symptom-card`, before the existing "Daily Symptom Log" card.

Insert this immediately after `<div class="symptom-section">`:

```html
      <div class="symptom-card">
        <div class="symptom-card-title">Flight Log <span class="info-pill">GA piloting</span></div>
        <button class="log-entry-btn" onclick="logFlight()">+ Log Flight</button>
        <div id="flight-log-list"></div>
      </div>

```

- [ ] **Step 2: Render the flight log on initial page load**

Locate the `restoreSymptoms` IIFE near the end of the `<script>` block (around line 1531). Inside that IIFE, the existing line `renderAllSymptomLogs();` already runs on init. Add a sibling call to `renderFlightLog()` immediately after it.

Find:

```javascript
  renderAllSymptomLogs();
})();
```

Change to:

```javascript
  renderAllSymptomLogs();
  renderFlightLog();
})();
```

- [ ] **Step 3: Reload and verify the Flight Log card renders**

Hard reload. Click the Symptoms tab. Verify:
- A "Flight Log" card appears at the top of the tab, above "Daily Symptom Log."
- The card shows the title, "GA piloting" pill, and a "+ Log Flight" button.
- No entries below the button (since none logged yet).

- [ ] **Step 4: Verify logging a flight**

Click "+ Log Flight." Verify:
- Toast appears: `✓ Flight logged`.
- An entry row appears below the button: `Flight | HH:MM AM | ×`.

Click "+ Log Flight" again (simulating a second flight). Verify:
- A second entry appears, newest on top.
- DevTools → localStorage → today's record contains `flightLog: [{at: "..."}, {at: "..."}]` (two entries).

- [ ] **Step 5: Verify entry removal**

Click the `×` on the first (most recent) entry. Verify:
- Toast: `Entry removed`.
- That entry disappears from the list.
- The other entry remains.
- localStorage `flightLog` now contains exactly 1 entry.

- [ ] **Step 6: Verify persistence across reload**

Hard reload. Click the Symptoms tab. Verify:
- The remaining flight entry is still displayed.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(symptoms): add flight log card with timestamped entries"
```

---

## Task 5: Render meals and flights in History view

**Files:**
- Modify: `index.html` — `renderHistory()` function (around lines 1320–1380)

- [ ] **Step 1: Update `renderHistory` to include meals and flight summary**

Find the `renderHistory` function. Locate the section where the per-day card HTML is built (the `keys.forEach(key => { ... })` block). The existing code reads:

```javascript
    const bristolLog = entry.bristolLog || [];
    const bristolSummary = bristolLog.length > 0
      ? bristolLog.map(b => `Type ${b.type} <span style="color:var(--text3)">${fmtTime(b.at)}</span>`).join(' · ')
      : null;
    html += `
      <div class="history-entry">
        <div class="history-date">
          <span>${dateStr}</span>
          <span class="history-compliance">${s.compliance ?? '—'}% compliance</span>
        </div>
        <div class="history-metrics" style="grid-template-columns:repeat(3,1fr)">
          ...
        </div>
        ${bristolSummary ? `<div class="history-note" style="border-top:1px solid var(--border);padding-top:8px;margin-top:4px;font-style:normal"><span style="color:var(--text3);font-size:10px;letter-spacing:1px;text-transform:uppercase">Bristol · </span>${bristolSummary}</div>` : ''}
        ${s.savedAt ? `<div style="font-size:10px;color:var(--text3);text-align:right;margin-top:6px">Saved ${fmtTime(s.savedAt)}</div>` : ''}
        ${s.notes ? `<div class="history-note">${s.notes}</div>` : ''}
      </div>`;
```

Replace it with (changes: add `meals` and `flights` derivations above; add two new lines into the template before the `Saved …` line):

```javascript
    const bristolLog = entry.bristolLog || [];
    const bristolSummary = bristolLog.length > 0
      ? bristolLog.map(b => `Type ${b.type} <span style="color:var(--text3)">${fmtTime(b.at)}</span>`).join(' · ')
      : null;

    const meals = entry.meals || {};
    const mealOrder = ['breakfast', 'lunch', 'dinner'];
    const mealLabels = { breakfast: 'Breakfast', lunch: 'Lunch', dinner: 'Dinner' };
    const mealLines = mealOrder
      .filter(m => meals[m] && meals[m].text)
      .map(m => {
        const safeText = meals[m].text.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
        return `<span class="history-meal-line"><span class="history-meal-name">${mealLabels[m]}</span><span class="history-meal-time">${fmtTime(meals[m].at)}</span>${safeText}</span>`;
      })
      .join('');

    const flightLog = entry.flightLog || [];
    const flightLine = flightLog.length > 0
      ? `<div class="history-flights"><span class="history-flights-label">✈ ${flightLog.length} flight${flightLog.length === 1 ? '' : 's'} ·</span>${flightLog.map(f => fmtTime(f.at)).join(', ')}</div>`
      : '';

    html += `
      <div class="history-entry">
        <div class="history-date">
          <span>${dateStr}</span>
          <span class="history-compliance">${s.compliance ?? '—'}% compliance</span>
        </div>
        <div class="history-metrics" style="grid-template-columns:repeat(3,1fr)">
          <div class="history-metric">
            <span class="metric-label">Bloat</span>
            <span class="metric-value">${s.bloating ?? '—'}</span>
          </div>
          <div class="history-metric">
            <span class="metric-label">Gas Pain</span>
            <span class="metric-value">${s.pain ?? '—'}</span>
          </div>
          <div class="history-metric">
            <span class="metric-label">Fatigue</span>
            <span class="metric-value">${s.fatigue ?? '—'}</span>
          </div>
          <div class="history-metric">
            <span class="metric-label">Tightness</span>
            <span class="metric-value">${s.tightness ?? '—'}</span>
          </div>
          <div class="history-metric">
            <span class="metric-label">Overall</span>
            <span class="metric-value">${s.overall ?? '—'}</span>
          </div>
          <div class="history-metric">
            <span class="metric-label">Bristol</span>
            <span class="metric-value">${bristolLog.length || '—'}</span>
          </div>
        </div>
        ${bristolSummary ? `<div class="history-note" style="border-top:1px solid var(--border);padding-top:8px;margin-top:4px;font-style:normal"><span style="color:var(--text3);font-size:10px;letter-spacing:1px;text-transform:uppercase">Bristol · </span>${bristolSummary}</div>` : ''}
        ${mealLines ? `<div class="history-meals"><span class="history-meals-label">Meals</span>${mealLines}</div>` : ''}
        ${flightLine}
        ${s.savedAt ? `<div style="font-size:10px;color:var(--text3);text-align:right;margin-top:6px">Saved ${fmtTime(s.savedAt)}</div>` : ''}
        ${s.notes ? `<div class="history-note">${s.notes}</div>` : ''}
      </div>`;
```

Note the **only** structural changes inside the existing template are the two new conditional lines:
- `${mealLines ? `<div class="history-meals">…</div>` : ''}`
- `${flightLine}`

Everything else above (Bristol, metrics, date header) is unchanged from the existing code — re-pasted only for surrounding context.

- [ ] **Step 2: Seed test data and verify history rendering**

In the DevTools console, set up a yesterday + today record so history has rows to display:

```javascript
const today = (() => { const n = new Date(); return `${n.getFullYear()}-${String(n.getMonth()+1).padStart(2,'0')}-${String(n.getDate()).padStart(2,'0')}`; })();
const yesterday = (() => { const n = new Date(); n.setDate(n.getDate() - 1); return `${n.getFullYear()}-${String(n.getMonth()+1).padStart(2,'0')}-${String(n.getDate()).padStart(2,'0')}`; })();
const d = JSON.parse(localStorage.sibo_tracker_v2 || '{}');
d[yesterday] = {
  checks: {},
  symptoms: { bloating: 2, pain: 1, fatigue: 3, tightness: 2, overall: 4, notes: 'Test day', compliance: 85, savedAt: new Date().toISOString() },
  meals: {
    breakfast: { text: 'eggs, toast, coffee', at: new Date(Date.now() - 86400000 + 8.25*3600*1000).toISOString() },
    lunch:     { text: 'chicken salad', at: new Date(Date.now() - 86400000 + 13*3600*1000).toISOString() },
    dinner:    { text: 'salmon, rice', at: new Date(Date.now() - 86400000 + 18.5*3600*1000).toISOString() }
  },
  flightLog: [
    { at: new Date(Date.now() - 86400000 + 9.25*3600*1000).toISOString() },
    { at: new Date(Date.now() - 86400000 + 15.5*3600*1000).toISOString() }
  ],
  bristolLog: []
};
localStorage.sibo_tracker_v2 = JSON.stringify(d);
```

Reload. Click the History tab. Verify the yesterday card shows:
- Date header with `85% compliance`.
- 6-metric grid (Bloat 2, Gas Pain 1, Fatigue 3, Tightness 2, Overall 4, Bristol —).
- A "Meals" line showing all three meals with times.
- A "✈ 2 flights · …, …" line.
- The "Test day" note.

- [ ] **Step 3: Verify partial-meal day renders only filled meals**

In the console:

```javascript
const d = JSON.parse(localStorage.sibo_tracker_v2);
delete d[yesterday].meals.lunch;
delete d[yesterday].meals.dinner;
localStorage.sibo_tracker_v2 = JSON.stringify(d);
```

Reload, History tab. Verify the yesterday card now shows only the Breakfast meal line (no Lunch, no Dinner), and the "Meals" section is still visible.

- [ ] **Step 4: Verify a no-meals/no-flights day omits both sections**

In the console:

```javascript
const d = JSON.parse(localStorage.sibo_tracker_v2);
delete d[yesterday].meals;
delete d[yesterday].flightLog;
localStorage.sibo_tracker_v2 = JSON.stringify(d);
```

Reload. Verify the yesterday card shows no "Meals" line and no "✈ flights" line. The "Test day" note is still present.

- [ ] **Step 5: Clean up the seeded test data**

```javascript
const d = JSON.parse(localStorage.sibo_tracker_v2);
delete d[yesterday];
localStorage.sibo_tracker_v2 = JSON.stringify(d);
```

Reload — confirm yesterday's card is gone from History.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(history): render meals and flight summary in per-day cards"
```

---

## Task 6: Bump version and verify JSON export

**Files:**
- Modify: `index.html` — version display (line 764)

- [ ] **Step 1: Bump the version pill from `v1.4` to `v1.5`**

Locate line 764:

```html
<div class="app-subtitle">Treatment Tracker <span style="opacity:0.6;font-size:10px;letter-spacing:1px">v1.4</span></div>
```

Change to:

```html
<div class="app-subtitle">Treatment Tracker <span style="opacity:0.6;font-size:10px;letter-spacing:1px">v1.5</span></div>
```

- [ ] **Step 2: End-to-end smoke test**

Hard reload. Perform a full real-world test pass:

1. **Today tab:** type "test breakfast" in the Breakfast food field, blur. Type "test lunch" in Lunch food field, blur. Type "test dinner" in Dinner food field, blur. Verify three "Logged HH:MM" lines appear.
2. **Symptoms tab:** click "+ Log Flight" twice. Verify two flight entries appear, newest first.
3. **Symptoms tab:** set Bloating to 2, Pain to 1, click "Save Symptom Log". Verify symptoms save toast.
4. **History tab:** verify today's card now shows the meals lines, the "✈ 2 flights" line, and the symptoms grid.

- [ ] **Step 3: Verify JSON export includes new fields**

While on the History tab, click "↓ Export Data (JSON)." A file `sibo-tracker-YYYY-MM-DD.json` downloads.

Open the downloaded JSON. Verify today's record contains:

```json
"YYYY-MM-DD": {
  "checks": {...},
  "symptoms": {...},
  "meals": {
    "breakfast": { "text": "test breakfast", "at": "..." },
    "lunch":     { "text": "test lunch",     "at": "..." },
    "dinner":    { "text": "test dinner",    "at": "..." }
  },
  "flightLog": [
    { "at": "..." },
    { "at": "..." }
  ],
  ...
}
```

If any new field is missing, return to the relevant prior task. Do not proceed.

- [ ] **Step 4: Clean up the test data**

In DevTools console:

```javascript
const k = (() => { const n = new Date(); return `${n.getFullYear()}-${String(n.getMonth()+1).padStart(2,'0')}-${String(n.getDate()).padStart(2,'0')}`; })();
const d = JSON.parse(localStorage.sibo_tracker_v2);
if (d[k]) {
  delete d[k].meals;
  delete d[k].flightLog;
  // Leave other fields (checks, symptoms) alone — they may have been pre-existing.
  localStorage.sibo_tracker_v2 = JSON.stringify(d);
}
```

Reload — confirm today's food fields and flight log are empty.

- [ ] **Step 5: Verify Reset Day still preserves observational data**

If today's record has any food entries or flight log entries, the next test won't be meaningful — re-add some test data:

```javascript
saveMeal('breakfast', 'reset-test breakfast');
logFlight();
```

Reload. On the Today tab, scroll to the Reset Day button (the floating one — actually, looking at the existing code, there is no visible Reset Day button placed in the DOM. Skip this verification step if no UI control exists.)

If a Reset Day control is present, click it, confirm. Verify:
- Medication checkboxes are cleared.
- Breakfast food field still shows "reset-test breakfast" — NOT cleared.
- Symptoms tab → Flight Log entry still present — NOT cleared.

If no Reset Day control is in the DOM (the existing code defines `resetDay` but nothing calls it from the UI), this verification can be done in the console:

```javascript
resetDay();
// confirm prompt → click Reset
```

Verify checks cleared but `meals` and `flightLog` still present in localStorage.

Then clean up:

```javascript
const d = JSON.parse(localStorage.sibo_tracker_v2);
const k = (() => { const n = new Date(); return `${n.getFullYear()}-${String(n.getMonth()+1).padStart(2,'0')}-${String(n.getDate()).padStart(2,'0')}`; })();
if (d[k]) { delete d[k].meals; delete d[k].flightLog; localStorage.sibo_tracker_v2 = JSON.stringify(d); }
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: meal and flight logging (v1.5)"
```

- [ ] **Step 7: Final review of git log**

```bash
git log --oneline -10
```

Expected commits (most recent first):
- `feat: meal and flight logging (v1.5)`
- `feat(history): render meals and flight summary in per-day cards`
- `feat(symptoms): add flight log card with timestamped entries`
- `feat(today): add per-meal food textarea with save-on-blur`
- `feat(data): add meal and flight-log persistence helpers`
- `feat(css): add styles for food block and history meal/flight rows`
- `spec: meal and flight logging design`
- `v1.4: fix day key to use local date (was UTC, caused nightly reset at 8pm EDT)`
- `remove IBgard from pre-breakfast block`
- `v1.3: timestamped entries for all symptoms, add abdominal tightness`

- [ ] **Step 8: Stop the local server**

Stop the `python3 -m http.server 8000` process (Ctrl-C in its terminal, or kill the background task).

---

## Out of Scope (do NOT implement)

These were explicitly deferred in the spec:
- Dedicated food-only History view.
- Multiple meal entries per meal slot (single editable entry per meal is sufficient).
- Flight metadata (duration, conditions, IFR/VFR, solo/dual).
- Additional day flags beyond flying.
- Server-side storage / sync.
