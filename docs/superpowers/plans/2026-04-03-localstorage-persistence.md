# LocalStorage Form Persistence — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Persist all form state in `localStorage` so accidental refreshes don't lose data.

**Architecture:** Single `localStorage` key stores radios, checkboxes, and text inputs as JSON. A delegated `change` listener saves on every interaction. Restore runs after DOM render. A "Clear Form" link resets everything. All changes are inline in `index.html`.

**Tech Stack:** Vanilla JS, localStorage API

**Spec:** `docs/superpowers/specs/2026-04-03-localstorage-persistence-design.md`

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Modify | `index.html:248-252` | Add "Clear Form" link to client bar HTML |
| Modify | `index.html:275` | Add restore toast element after existing `#copyToast` |
| Modify | `index.html:1108` (before `</script>`) | Add `saveState()`, `restoreState()`, `clearForm()` JS functions + event listeners |

---

### Task 1: Add "Clear Form" Link and Restore Toast HTML

**Files:**
- Modify: `index.html:248-252` (client bar)
- Modify: `index.html:275` (after copyToast)

- [ ] **Step 1: Add "Clear Form" link to client bar**

In `index.html`, find the closing `</div>` of the `.client-bar` (after the 3 field divs at lines 249-251). Add a Clear Form link as the last child of `.client-bar`:

```html
<div class="client-bar">
  <div class="field"><label>Client Name</label><input type="text" placeholder="Full name"></div>
  <div class="field"><label>Date of Birth</label><input type="text" placeholder="MM / DD / YYYY" style="min-width:130px"></div>
  <div class="field"><label>Date Completed</label><input type="text" placeholder="MM / DD / YYYY" style="min-width:130px"></div>
  <a href="#" onclick="clearForm();return false" style="font-size:0.78rem;color:var(--k-muted);text-decoration:underline;white-space:nowrap;margin-left:auto">Clear Form</a>
</div>
```

- [ ] **Step 2: Add restore toast element**

In `index.html`, find the existing `#copyToast` div (line 275). Add a new toast element immediately after it:

```html
<div id="restoreToast" style="display:none;position:fixed;bottom:80px;right:24px;background:var(--k-dark);color:white;padding:10px 20px;border-radius:8px;font-size:0.85rem;z-index:300">↻ Previous answers restored</div>
```

- [ ] **Step 3: Verify in browser**

Open `http://localhost:8765/index.html`. Check:
- "Clear Form" link appears at the right end of the client bar
- It's subtle (small, muted color, underlined)
- At 375px mobile, it wraps with the other fields (the existing `flex-wrap: wrap` on `.client-bar` handles this)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Clear Form link and restore toast HTML"
```

---

### Task 2: Implement saveState() Function

**Files:**
- Modify: `index.html` — add JS before the closing `</script>` tag (currently line 1109)

- [ ] **Step 1: Add the saveState function**

Insert the following after the `copyScores` function (after line 1108, before `</script>`):

```javascript
// ─── LOCAL STORAGE PERSISTENCE ─────────────────────────────────────────
var STORAGE_KEY = 'kh_assessment_state';

function saveState() {
  var radios = {};
  document.querySelectorAll('#form-root input[type="radio"]:checked').forEach(function(r) {
    radios[r.name] = r.value;
  });
  var checkboxes = {};
  document.querySelectorAll('#form-root .exp-row').forEach(function(row, ri) {
    row.querySelectorAll('input[type="checkbox"]').forEach(function(cb, ci) {
      if (cb.checked) checkboxes['exp_' + ri + '_' + ci] = true;
    });
  });
  var textInputs = [];
  document.querySelectorAll('.client-bar .field input').forEach(function(inp) {
    textInputs.push(inp.value);
  });
  localStorage.setItem(STORAGE_KEY, JSON.stringify({
    radios: radios,
    checkboxes: checkboxes,
    textInputs: textInputs,
    savedAt: new Date().toISOString()
  }));
}
```

- [ ] **Step 2: Attach event listeners for saving**

Add immediately after the `saveState` function:

```javascript
document.getElementById('form-root').addEventListener('change', saveState);
document.querySelectorAll('.client-bar .field input').forEach(function(inp) {
  inp.addEventListener('input', saveState);
});
```

- [ ] **Step 3: Verify save works**

Open browser, open DevTools Console. Click a radio button in the first section. Run:

```javascript
JSON.parse(localStorage.getItem('kh_assessment_state'))
```

Expected: an object with `radios` containing the clicked radio's name and value.

Type in the Client Name field. Re-run the command. Expected: `textInputs[0]` contains the typed text.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add saveState() with change listeners for localStorage persistence"
```

---

### Task 3: Implement restoreState() Function

**Files:**
- Modify: `index.html` — add JS after the save event listeners from Task 2

- [ ] **Step 1: Add the restoreState function**

Add after the save event listeners:

```javascript
function restoreState() {
  var raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return;
  var state;
  try { state = JSON.parse(raw); } catch(e) { return; }

  if (state.radios) {
    Object.keys(state.radios).forEach(function(name) {
      var radio = document.querySelector('input[name="' + name + '"][value="' + state.radios[name] + '"]');
      if (radio) radio.checked = true;
    });
  }

  if (state.checkboxes) {
    document.querySelectorAll('#form-root .exp-row').forEach(function(row, ri) {
      row.querySelectorAll('input[type="checkbox"]').forEach(function(cb, ci) {
        if (state.checkboxes['exp_' + ri + '_' + ci]) cb.checked = true;
      });
    });
  }

  if (state.textInputs) {
    document.querySelectorAll('.client-bar .field input').forEach(function(inp, i) {
      if (state.textInputs[i] != null) inp.value = state.textInputs[i];
    });
  }

  SECTIONS.forEach(function(sec) { recalc(sec.id); });

  var toast = document.getElementById('restoreToast');
  if (toast) { toast.style.display = 'block'; setTimeout(function() { toast.style.display = 'none'; }, 3000); }
}

restoreState();
```

Note: `restoreState()` is called immediately at script load time. This works because the `<script>` block is at the bottom of the `<body>`, after the `DOMContentLoaded` handler has already rendered the form (the form render is synchronous inside `DOMContentLoaded`, which fires before the script continues execution past its registration). Actually — the form is rendered *inside* `DOMContentLoaded`, and this call happens at the module level. We need to call `restoreState()` at the end of the `DOMContentLoaded` handler instead.

**Correction:** Do NOT add `restoreState();` after the function definition. Instead, add the call at the end of the existing `DOMContentLoaded` handler, after line 1053 (`console.log('✅ Katalyst Assessment loaded:'...)`), inside the `try` block:

```javascript
    console.log('✅ Katalyst Assessment loaded:', SECTIONS.length, 'sections');
    restoreState();
```

- [ ] **Step 2: Verify restore works**

1. Open browser, fill in a few radio buttons and type a client name
2. Refresh the page
3. Expected: all selections and text are restored, scores recalculate, "Previous answers restored" toast appears for 3 seconds

- [ ] **Step 3: Verify restore with no saved data**

1. Open DevTools Console, run `localStorage.removeItem('kh_assessment_state')`
2. Refresh the page
3. Expected: form loads empty, no toast appears

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add restoreState() to recover form data from localStorage on load"
```

---

### Task 4: Implement clearForm() Function

**Files:**
- Modify: `index.html` — add JS after `restoreState` function

- [ ] **Step 1: Add the clearForm function**

Add after the `restoreState` function (before the closing `</script>`):

```javascript
window.clearForm = function() {
  localStorage.removeItem(STORAGE_KEY);
  location.reload();
};
```

- [ ] **Step 2: Verify clear works**

1. Open browser, fill in some answers
2. Click "Clear Form" in the client bar
3. Expected: page reloads with a completely empty form, no toast, no saved data

- [ ] **Step 3: Verify localStorage is actually cleared**

Open DevTools Console after clicking Clear Form:

```javascript
localStorage.getItem('kh_assessment_state')
```

Expected: `null`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add clearForm() to wipe localStorage and reset the assessment"
```

---

### Task 5: Final Verification and Push

**Files:** None modified — verification only.

- [ ] **Step 1: Full round-trip test**

1. Open `http://localhost:8765/index.html`
2. Fill in client name: "Test User"
3. Fill in DOB: "01/01/1990"
4. Answer 3-4 radio questions across different sections
5. Check a checkbox in an exposure section (if visible)
6. Verify scores update correctly
7. Refresh the page
8. Expected: all answers restored, scores recalculated, toast shown

- [ ] **Step 2: Test at mobile (375px)**

1. Resize to 375x812
2. Verify "Clear Form" link is visible and accessible
3. Fill in a couple of answers, refresh, verify restore works
4. Click "Clear Form", verify form resets

- [ ] **Step 3: Test corrupted localStorage**

In DevTools Console:
```javascript
localStorage.setItem('kh_assessment_state', 'not valid json');
```
Refresh the page. Expected: form loads empty, no errors in console.

- [ ] **Step 4: Push**

```bash
git push origin master
```
