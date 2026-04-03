# Multi-Step Wizard Pagination — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Break the single-page 22-section assessment into an 8-step wizard (7 groups + Review & Print) to fix rendering failures and improve mobile usability.

**Architecture:** All 22 section cards stay in the DOM. The render loop wraps each group in a `.wizard-step` div. A `goToStep(n)` function toggles visibility via `display:none`. Step 8 shows everything for print. Existing scoring, localStorage, and print CSS are unchanged.

**Tech Stack:** Vanilla JS, CSS, inline in `index.html`

**Spec:** `docs/superpowers/specs/2026-04-03-wizard-pagination-design.md`

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Modify | `index.html:146` (after `.fab-copy:hover`) | Add wizard CSS (step bar, nav bar, step visibility) |
| Modify | `index.html:195` (`@media print`) | Add print overrides for wizard elements |
| Modify | `index.html:217` (after `</header>`) | Insert wizard header HTML (step dots) |
| Modify | `index.html:255` (after `</div><!-- form-root -->`) | Move summary card into form-root area for step 8 |
| Modify | `index.html:275` (before `<script>`) | Insert wizard nav HTML (bottom bar) |
| Modify | `index.html:960-1053` (render loop) | Wrap sections in `.wizard-step` containers per group |
| Modify | `index.html:1055-1056` (after render) | Call `goToStep(1)` after render + restore |
| Modify | `index.html:1113-1175` (localStorage) | Add `currentStep` to save/restore, add `goToStep()` function |

---

### Task 1: Add Wizard CSS

**Files:**
- Modify: `index.html` — insert new CSS rules after `.fab-copy:hover` (line 145) and before the `@media` blocks

- [ ] **Step 1: Add wizard CSS rules**

Insert the following CSS block in `index.html` between the `.fab-copy:hover` rule (line 145) and the blank line before `@media screen and (max-width: 600px)` (line 147):

```css

.wizard-header{text-align:center;padding:16px 20px 8px;background:white;border-bottom:1px solid var(--k-rule)}
.step-dots{display:flex;justify-content:center;gap:8px;margin-bottom:6px}
.step-dot{width:32px;height:32px;border-radius:50%;display:inline-flex;align-items:center;justify-content:center;font-size:0.8rem;font-weight:600;font-family:'Inter',sans-serif;transition:all 0.2s}
.step-dot.completed{background:var(--k-mid);color:white;cursor:pointer}
.step-dot.completed:hover{background:var(--k-dark)}
.step-dot.current{background:white;border:2px solid var(--k-mid);color:var(--k-mid)}
.step-dot.future{background:#e0e0e0;color:#999}
.step-label{font-size:0.82rem;color:var(--k-muted);font-family:'Inter',sans-serif}
.wizard-step{display:none}
.wizard-step.active{display:block}
.wizard-nav{position:fixed;bottom:0;left:0;right:0;background:white;border-top:2px solid var(--k-rule);padding:12px 24px;display:flex;justify-content:space-between;align-items:center;z-index:150;font-family:'Inter',sans-serif}
.wizard-nav button{padding:12px 28px;border-radius:8px;font-size:0.9rem;font-weight:600;cursor:pointer;transition:all 0.2s}
.wiz-back{background:white;border:1px solid #ccc;color:var(--k-dark)}
.wiz-back:hover{border-color:var(--k-dark)}
.wiz-next{background:var(--k-mid);color:white;border:none}
.wiz-next:hover{background:var(--k-dark)}
.page-footer{margin-bottom:70px}
.summary-wrap{display:none}
.summary-wrap.active{display:block}
```

- [ ] **Step 2: Add wizard-specific responsive rules**

In the existing `@media screen and (max-width: 600px)` block (currently around line 147), add these rules at the end of that block, before the closing `}`:

```css
  .wizard-nav { padding: 10px 12px; }
  .wizard-nav button { padding: 10px 20px; font-size: 0.82rem; }
  .step-dot { width: 28px; height: 28px; font-size: 0.72rem; }
```

- [ ] **Step 3: Add mobile dot collapse**

Add a new media query after the existing `@media screen and (max-width: 480px)` block and before `@media print`:

```css
@media screen and (max-width: 400px) {
  .step-dots { display: none; }
}
```

- [ ] **Step 4: Add print overrides for wizard elements**

In the existing `@media print` block, add these rules to ensure print shows everything and hides wizard chrome:

```css
  .wizard-header{display:none!important}
  .wizard-nav{display:none!important}
  .wizard-step{display:block!important}
  .summary-wrap{display:block!important}
```

- [ ] **Step 5: Verify CSS parses**

Open `http://localhost:8765/index.html` in browser. Open DevTools Console. Check for no CSS parse errors. The page will look broken (no wizard JS yet) — that's expected.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add wizard CSS for step bar, nav, and step visibility"
```

---

### Task 2: Add Wizard HTML (Step Bar + Nav Bar)

**Files:**
- Modify: `index.html` — insert HTML for wizard header and wizard nav

- [ ] **Step 1: Insert wizard header after page header**

In `index.html`, find the closing `</header>` tag (line 217). Insert the wizard header HTML immediately after it, before `<div class="companion-bar">`:

```html
<div class="wizard-header" id="wizard-header">
  <div class="step-dots" id="step-dots"></div>
  <div class="step-label" id="step-label"></div>
</div>
```

The step dots will be populated by JS in Task 3 (the group names come from the `SECTIONS` data).

- [ ] **Step 2: Wrap the summary card in a wizard-step container**

Find the summary card HTML (currently lines 257-265):

```html
<div class="main-wrap" style="padding-top:0">
  <div class="summary-card">
    ...
  </div>
</div>
```

Wrap it with a `summary-wrap` div by replacing it with:

```html
<div class="summary-wrap" id="summary-wrap">
<div class="main-wrap" style="padding-top:0">
  <div class="summary-card">
    <div class="sec-head"><h3>Score Summary — All Sections</h3><div class="sec-head-badge">Complete the form above first</div></div>
    <table class="sum-table" id="sum-table">
      <thead><tr><th style="width:36%">Section</th><th style="width:10%">Score</th><th style="width:14%">Range</th><th>🟢 Green</th><th>🟡 Yellow</th><th>🔴 Red</th></tr></thead>
      <tbody id="sum-body"></tbody>
    </table>
  </div>
</div>
</div>
```

- [ ] **Step 3: Insert wizard nav before the script tag**

Find the `restoreToast` div and the `<script>` tag. Insert the wizard nav HTML between the `restoreToast` div and `<script>`:

```html
<div class="wizard-nav" id="wizard-nav">
  <button class="wiz-back" id="wiz-back" onclick="goToStep(currentStep - 1)">← Back</button>
  <button class="wiz-next" id="wiz-next" onclick="goToStep(currentStep + 1)">Next →</button>
</div>
```

- [ ] **Step 4: Hide FAB buttons by default**

Add `style="display:none"` to both FAB buttons so they're hidden during steps 1-7 (the `goToStep` function will show them on step 8):

Change:
```html
<button class="fab-print" onclick="window.print()">🖨&nbsp; Print / Save as PDF</button>
<button class="fab-copy" onclick="copyScores()">📋&nbsp; Copy Score Summary</button>
```

To:
```html
<button class="fab-print" id="fab-print" onclick="window.print()" style="display:none">🖨&nbsp; Print / Save as PDF</button>
<button class="fab-copy" id="fab-copy" onclick="copyScores()" style="display:none">📋&nbsp; Copy Score Summary</button>
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add wizard header, nav bar, and summary wrapper HTML"
```

---

### Task 3: Modify Render Loop to Wrap Groups in Wizard Steps

**Files:**
- Modify: `index.html` — the `DOMContentLoaded` render loop (lines 960-1053)

- [ ] **Step 1: Add step counter and wrapper tracking**

Find the render loop setup (lines 960-961):

```javascript
    let curGroup = '';
```

Replace with:

```javascript
    let curGroup = '';
    let stepNum = 0;
    let stepWrap = null;
    var GROUP_NAMES = [];
```

- [ ] **Step 2: Modify the group-change block to create wizard-step wrappers**

Find the block that creates group titles (lines 962-969):

```javascript
    SECTIONS.forEach((sec) => {
      if (sec.group !== curGroup) {
        curGroup = sec.group;
        const gt = document.createElement('div');
        gt.className = 'group-title';
        gt.textContent = sec.group;
        root.appendChild(gt);
      }
```

Replace with:

```javascript
    SECTIONS.forEach((sec) => {
      if (sec.group !== curGroup) {
        curGroup = sec.group;
        stepNum++;
        GROUP_NAMES.push(sec.group);
        stepWrap = document.createElement('div');
        stepWrap.className = 'wizard-step';
        stepWrap.dataset.step = stepNum;
        root.appendChild(stepWrap);
        const gt = document.createElement('div');
        gt.className = 'group-title';
        gt.textContent = sec.group;
        stepWrap.appendChild(gt);
      }
```

- [ ] **Step 3: Change card appends from root to stepWrap**

Find the line that appends the card to root (line 1048):

```javascript
      root.appendChild(card);
```

Replace with:

```javascript
      stepWrap.appendChild(card);
```

- [ ] **Step 4: Build step dots after the render loop**

Find the lines after the render loop ends (lines 1053-1056):

```javascript
    });

    console.log('\u2705 Katalyst Assessment loaded:', SECTIONS.length, 'sections');
    restoreState();
```

Replace with:

```javascript
    });

    // Build step dots
    var dotsEl = document.getElementById('step-dots');
    GROUP_NAMES.forEach(function(name, i) {
      var dot = document.createElement('div');
      dot.className = 'step-dot future';
      dot.textContent = i + 1;
      dot.dataset.step = i + 1;
      dot.onclick = function() { if (this.classList.contains('completed')) goToStep(i + 1); };
      dotsEl.appendChild(dot);
    });
    // Add step 8 dot (Review & Print)
    var dot8 = document.createElement('div');
    dot8.className = 'step-dot future';
    dot8.textContent = '8';
    dot8.dataset.step = 8;
    dot8.onclick = function() { if (this.classList.contains('completed')) goToStep(8); };
    dotsEl.appendChild(dot8);

    console.log('\u2705 Katalyst Assessment loaded:', SECTIONS.length, 'sections');
    restoreState();
```

- [ ] **Step 5: Verify render**

Open `http://localhost:8765/index.html`. The page should show 8 grey dots at the top but no form content (all wizard steps are `display:none` by default). The wizard nav bar should appear at the bottom. This is expected — `goToStep()` doesn't exist yet.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: wrap form sections in wizard-step containers per group"
```

---

### Task 4: Implement goToStep() and Wire Up Navigation

**Files:**
- Modify: `index.html` — add `goToStep()` function and update localStorage functions

- [ ] **Step 1: Add goToStep function and currentStep variable**

Find the localStorage section header (line 1113):

```javascript
// ─── LOCAL STORAGE PERSISTENCE ─────────────────────────────────────────
var STORAGE_KEY = 'kh_assessment_state';
```

Insert the wizard navigation code **before** it:

```javascript
// ─── WIZARD NAVIGATION ─────────────────────────────────────────────────
var currentStep = 1;
var TOTAL_STEPS = 8;

window.goToStep = function(n) {
  if (n < 1 || n > TOTAL_STEPS) return;
  currentStep = n;

  // Hide all wizard-step containers
  document.querySelectorAll('.wizard-step').forEach(function(el) {
    el.classList.remove('active');
  });

  // Update step dots
  document.querySelectorAll('.step-dot').forEach(function(dot) {
    var s = parseInt(dot.dataset.step);
    if (s < n) {
      dot.className = 'step-dot completed';
      dot.textContent = '\u2713';
    } else if (s === n) {
      dot.className = 'step-dot current';
      dot.textContent = s;
    } else {
      dot.className = 'step-dot future';
      dot.textContent = s;
    }
  });

  var wizHeader = document.getElementById('wizard-header');
  var summaryWrap = document.getElementById('summary-wrap');
  var fabPrint = document.getElementById('fab-print');
  var fabCopy = document.getElementById('fab-copy');
  var backBtn = document.getElementById('wiz-back');
  var nextBtn = document.getElementById('wiz-next');
  var stepLabel = document.getElementById('step-label');

  if (n === 8) {
    // Step 8: Review & Print — show all steps + summary
    document.querySelectorAll('.wizard-step').forEach(function(el) {
      el.classList.add('active');
    });
    summaryWrap.classList.add('active');
    fabPrint.style.display = '';
    fabCopy.style.display = '';
    wizHeader.querySelector('.step-dots').style.display = 'none';
    stepLabel.textContent = 'Review & Print — All Sections';
    backBtn.textContent = '\u2190 Back to Form';
    backBtn.style.display = '';
    nextBtn.style.display = 'none';
  } else {
    // Steps 1-7: show only the current group
    var activeStep = document.querySelector('.wizard-step[data-step="' + n + '"]');
    if (activeStep) activeStep.classList.add('active');
    summaryWrap.classList.remove('active');
    fabPrint.style.display = 'none';
    fabCopy.style.display = 'none';
    wizHeader.querySelector('.step-dots').style.display = '';
    stepLabel.textContent = 'Step ' + n + ' of ' + TOTAL_STEPS + ' \u00b7 ' + (GROUP_NAMES[n - 1] || '');
    backBtn.textContent = '\u2190 Back';
    backBtn.style.display = (n === 1) ? 'none' : '';
    nextBtn.style.display = '';
    nextBtn.textContent = (n === 7) ? 'Review & Print \u2192' : 'Next \u2192';
  }

  window.scrollTo(0, 0);
  saveState();
};
```

- [ ] **Step 2: Update saveState to include currentStep**

Find the `localStorage.setItem` call in `saveState()` (lines 1129-1134):

```javascript
  localStorage.setItem(STORAGE_KEY, JSON.stringify({
    radios: radios,
    checkboxes: checkboxes,
    textInputs: textInputs,
    savedAt: new Date().toISOString()
  }));
```

Replace with:

```javascript
  localStorage.setItem(STORAGE_KEY, JSON.stringify({
    radios: radios,
    checkboxes: checkboxes,
    textInputs: textInputs,
    currentStep: currentStep,
    savedAt: new Date().toISOString()
  }));
```

- [ ] **Step 3: Update restoreState to restore currentStep**

Find the end of `restoreState()` where it shows the toast (lines 1161-1164):

```javascript
  SECTIONS.forEach(function(sec) { recalc(sec.id); });

  var toast = document.getElementById('restoreToast');
  if (toast) { toast.style.display = 'block'; setTimeout(function() { toast.style.display = 'none'; }, 3000); }
}
```

Replace with:

```javascript
  SECTIONS.forEach(function(sec) { recalc(sec.id); });

  if (state.currentStep) {
    goToStep(state.currentStep);
  } else {
    goToStep(1);
  }

  var toast = document.getElementById('restoreToast');
  if (toast) { toast.style.display = 'block'; setTimeout(function() { toast.style.display = 'none'; }, 3000); }
}
```

- [ ] **Step 4: Add goToStep(1) fallback after restoreState**

Find the `restoreState()` call at the end of `DOMContentLoaded` (in the code added in Task 3 Step 4). After `restoreState();`, add a fallback in case there was no saved state:

```javascript
    restoreState();
    if (!localStorage.getItem(STORAGE_KEY)) goToStep(1);
```

- [ ] **Step 5: Verify wizard works**

Open `http://localhost:8765/index.html`.

Expected:
- Step 1 is shown: "Immune History & Energy Systems" with 2 section cards
- Step dot 1 has green outline (current), dots 2-8 are grey
- Label says "Step 1 of 8 · Immune History & Energy Systems"
- Back button is hidden, Next button says "Next →"
- Click Next: step 2 shows "Nutrient & Thyroid Status" with 3 section cards
- Step dot 1 is green checkmark, dot 2 has outline
- Click Back: returns to step 1
- Navigate to step 7, click "Review & Print →": all sections visible, summary table shown, FAB buttons visible
- Click "← Back to Form": returns to step 7

- [ ] **Step 6: Verify localStorage**

1. Navigate to step 4
2. Refresh the page
3. Expected: restores to step 4

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add goToStep() wizard navigation with localStorage step tracking"
```

---

### Task 5: Final Verification and Push

**Files:** None modified — verification only.

- [ ] **Step 1: Full wizard walkthrough at desktop (1280px)**

Navigate through all 8 steps. At each step:
- Correct sections are shown
- Step dots update correctly (completed = checkmark, current = outline)
- Back/Next buttons have correct labels
- Step 8 shows all sections + summary table + FAB buttons

- [ ] **Step 2: Test scoring across steps**

1. On step 1, answer a few radio buttons in Mitochondrial Dysfunction
2. Navigate to step 4, answer some radio buttons
3. Go to step 8 (Review & Print)
4. Check that score summary table shows correct scores for both sections
5. Click "Copy Score Summary" — verify it copies all section scores

- [ ] **Step 3: Test at mobile (375px)**

1. Resize to 375x812
2. Navigate through steps — confirm wizard nav buttons are accessible
3. On step 8, scroll through all sections — confirm layout works
4. Print preview — confirm all sections appear

- [ ] **Step 4: Test at 400px (dot collapse)**

1. Resize to 390x812
2. Confirm step dots are hidden and only the text label shows

- [ ] **Step 5: Test print from step 8**

1. Navigate to step 8
2. Click "Print / Save as PDF"
3. Expected: print preview shows all sections, no wizard header or nav bar, clean layout matching original

- [ ] **Step 6: Test localStorage persistence**

1. Fill in client name, answer some questions on steps 1 and 3
2. Navigate to step 5
3. Refresh page
4. Expected: returns to step 5, client name and answers restored

- [ ] **Step 7: Test Clear Form**

1. Click "Clear Form" in client bar
2. Expected: reloads to step 1, all answers cleared

- [ ] **Step 8: Push**

```bash
git push origin master
```
