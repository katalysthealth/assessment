# Multi-Step Wizard Pagination — Design Spec

## Goal

Break the 37,000+ pixel single-page assessment into an 8-step wizard to eliminate rendering failures, browser slowdowns, and lost scroll position. Steps 1-7 correspond to the 7 existing group titles. Step 8 is a "Review & Print" view that shows all answers consolidated for printing/PDF.

## Step Mapping

| Step | Group Title | Sections |
|------|------------|----------|
| 1 | Immune History & Energy Systems | 2 |
| 2 | Nutrient & Thyroid Status | 3 |
| 3 | HPA Axis & Autonomic Nervous System | 2 |
| 4 | Pathogens | 2 |
| 5 | Lyme & Co-Infections | 4 |
| 6 | Immune Dysregulation | 1 |
| 7 | Toxicants & Toxins | 8 |
| 8 | Review & Print | All (read-only consolidated view) |

## Architecture

All 22 section cards remain in the DOM at all times. The wizard works by toggling `display:none` on group containers — no re-rendering, no DOM manipulation beyond visibility. This preserves all existing scoring JS, localStorage persistence, and event listeners unchanged.

### New DOM Structure

```
<div class="wizard-header">        ← step dots + label (new)
<div class="client-bar">            ← unchanged, always visible
<div class="scale-legend">          ← unchanged, always visible
<div class="instr-banner">          ← unchanged, always visible
<div id="form-root">
  <div class="wizard-step" data-step="1">  ← wrapper per group (new)
    <div class="group-title">...</div>
    <div class="sec-card">...</div>
    <div class="sec-card">...</div>
  </div>
  <div class="wizard-step" data-step="2">
    ...
  </div>
  ... steps 3-7 ...
</div>
<div class="wizard-step" data-step="8" id="review-step">  ← review view (new)
  (all sections visible, summary table, FAB buttons)
</div>
<div class="wizard-nav">            ← fixed bottom bar (new)
```

### Rendering Change

The existing `DOMContentLoaded` handler renders sections into `#form-root` sequentially. The change is to wrap each group's sections in a `<div class="wizard-step" data-step="N">` container. When the current group title changes during the render loop, close the previous wrapper and open a new one.

## Step Bar

- Row of 8 numbered circles at the top, below the page header and above the client bar
- **Completed steps:** filled green with a checkmark, clickable (jumps to that step for review/editing)
- **Current step:** white with green outline and bold number, not clickable
- **Future steps:** grey fill, grey number, not clickable
- Below the dots: text label "Step N of 8 · {Group Title}"
- On mobile (<400px): dots collapse to text "Step 4 / 8 · Pathogens" (no circles)

### Step Bar CSS

```css
.wizard-header { text-align: center; padding: 16px 20px 8px; background: white; border-bottom: 1px solid var(--k-rule); }
.step-dots { display: flex; justify-content: center; gap: 8px; margin-bottom: 6px; }
.step-dot { width: 32px; height: 32px; border-radius: 50%; display: inline-flex; align-items: center; justify-content: center; font-size: 0.8rem; font-weight: 600; font-family: 'Inter', sans-serif; transition: all 0.2s; }
.step-dot.completed { background: var(--k-mid); color: white; cursor: pointer; }
.step-dot.current { background: white; border: 2px solid var(--k-mid); color: var(--k-mid); }
.step-dot.future { background: #e0e0e0; color: #999; }
.step-label { font-size: 0.82rem; color: var(--k-muted); font-family: 'Inter', sans-serif; }

@media screen and (max-width: 400px) {
  .step-dots { display: none; }
}
```

## Bottom Navigation Bar

- Fixed to the bottom of the viewport (above the existing FAB buttons area)
- Contains Back and Next buttons
- Back is hidden on step 1
- On step 7, Next reads "Review & Print →"
- On step 8, Next is hidden; the FAB buttons (Print/PDF, Copy Summary) are shown instead
- Clicking Next scrolls the viewport to the top of the page

### Button States

| Step | Back | Next |
|------|------|------|
| 1 | Hidden | "Next →" |
| 2-6 | "← Back" | "Next →" |
| 7 | "← Back" | "Review & Print →" |
| 8 | "← Back to Form" | Hidden (FABs shown instead) |

### Nav Bar CSS

```css
.wizard-nav { position: fixed; bottom: 0; left: 0; right: 0; background: white; border-top: 2px solid var(--k-rule); padding: 12px 24px; display: flex; justify-content: space-between; z-index: 150; font-family: 'Inter', sans-serif; }
.wizard-nav button { padding: 12px 28px; border-radius: 8px; font-size: 0.9rem; font-weight: 600; cursor: pointer; transition: all 0.2s; }
.wiz-back { background: white; border: 1px solid #ccc; color: var(--k-dark); }
.wiz-next { background: var(--k-mid); color: white; border: none; }
.wiz-next:hover { background: var(--k-dark); }
```

## Step 8 — Review & Print

When entering step 8:
- Remove `display:none` from all `.wizard-step` containers (show all sections)
- Show the summary table (`.summary-card`)
- Show the FAB buttons (Print/PDF, Copy Summary)
- Hide the step bar dots (replace with "Review & Print — All Sections" heading)
- Page becomes scrollable to view all answers

When leaving step 8 (clicking "← Back to Form"):
- Re-hide all steps except step 7
- Restore the step bar
- Hide FAB buttons

Print styles remain completely unchanged — `@media print` already hides interactive elements and formats the full page.

## FAB Button Changes

- Steps 1-7: FAB buttons (Print, Copy) are hidden — the wizard nav is the primary bottom UI
- Step 8: FAB buttons are shown in their current fixed position
- The `.page-footer` remains visible on all steps

## localStorage Changes

Add `currentStep` (integer 1-8) to the existing saved state object:

```json
{
  "radios": { ... },
  "checkboxes": { ... },
  "textInputs": [ ... ],
  "currentStep": 4,
  "savedAt": "..."
}
```

On restore, navigate to the saved step. Default to step 1 if absent.

## Page Scroll Behavior

- On step change (Next, Back, or dot click): `window.scrollTo(0, 0)` to scroll to top
- This ensures the user always starts at the top of each group's content

## Existing Features — Impact Assessment

| Feature | Impact |
|---------|--------|
| Scoring (`recalc`) | None — all radios remain in DOM, `recalc` queries by name as before |
| localStorage persistence | Minor — add `currentStep` to saved state |
| Copy Score Summary | None — queries all sections regardless of visibility |
| Print / Save as PDF | None — print styles override display, show everything |
| Mobile responsive CSS | None — existing `@media` blocks apply within each step's content |
| Client bar / scale legend | None — always visible above wizard |

## Scope

- All changes are inline in `index.html`
- Modify the render loop to wrap groups in `.wizard-step` containers
- Add wizard header HTML (step dots)
- Add wizard nav HTML (bottom bar)
- Add `goToStep(n)` function + step dot click handlers
- Modify `restoreState()` to restore `currentStep`
- Modify `saveState()` to save `currentStep`
- Add wizard CSS (step bar, nav bar, step visibility)
- Hide FABs on steps 1-7, show on step 8

## Out of Scope

- Animations or transitions between steps
- Per-step validation (all questions are optional)
- Step completion tracking (checkmarks based on "answered all questions in step")  — completed means "visited", not "answered"
- Keyboard navigation between steps
