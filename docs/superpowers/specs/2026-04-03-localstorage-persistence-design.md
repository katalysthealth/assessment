# LocalStorage Form Persistence — Design Spec

## Goal

Persist all form state in the browser's `localStorage` so that an accidental page refresh does not lose data. Single-form-at-a-time model — one assessment in progress, with a "Clear Form" action to reset for the next client.

## Storage Format

Single `localStorage` key: `kh_assessment_state`

```json
{
  "radios": { "mito_q1": "3", "drain_q0": "1", ... },
  "checkboxes": { "mold_q5_0": true, ... },
  "textInputs": ["Jane Doe", "01/15/1980", "04/03/2026"],
  "savedAt": "2026-04-03T22:15:00Z"
}
```

- `radios` — keyed by radio group `name` attribute, value is the selected radio's `value` string
- `checkboxes` — keyed by a generated identifier (section + question index + checkbox index within the row), value is boolean
- `textInputs` — array of 3 values in DOM order: client name, DOB, date completed
- `savedAt` — ISO timestamp for debugging; not used in restore logic

## Save Behavior

- Attach a single delegated `change` event listener on `#form-root` (covers radios and checkboxes) and `input` listeners on the 3 text fields in `.client-bar`
- On any change, serialize current form state and write to `localStorage`
- No debounce needed — `localStorage.setItem` is synchronous and fast

## Restore Behavior

- Runs once at end of `DOMContentLoaded`, after the form has been rendered
- Read `kh_assessment_state` from `localStorage`; if absent or unparseable, do nothing
- For each saved radio: find `input[name="..."][value="..."]`, set `.checked = true`
- For each saved checkbox: find by generated identifier, set `.checked` accordingly
- For each text input: set `.value` from the saved array
- Call `recalc(sectionId)` for every section to update all scores and color coding
- Show a toast "Previous answers restored" for 3 seconds (reuse the `#copyToast` element's pattern/styling)

## Clear Form

- Add a "Clear Form" text link/button in the `.client-bar` area
- On click: call `localStorage.removeItem('kh_assessment_state')` and `location.reload()`
- Style as a subtle text link so it doesn't compete with the primary FAB buttons

## Scope

- All changes are inline in `index.html` — no new files
- ~40 lines of JS added after the existing `recalc` and `copyScores` functions
- One small HTML addition for the "Clear Form" link in the client bar
- One new toast element for the restore notification (or reuse existing `#copyToast`)

## Out of Scope

- Multi-client / named sessions
- Server-side persistence
- Auto-expiry of saved data
- Export/import of saved state
