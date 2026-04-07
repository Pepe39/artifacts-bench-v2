# Command: fill

## Purpose
Given a completed `rubrics.md` for a task, fill the Feather form: each rubric Pass or Fail with rationale, overall rating with rationale, and the "rubrics sufficient" answer. Never submits.

## Parameters
- `$ARGUMENTS`: UUID (required). Reads from `tasks/*/<UUID>/rubrics.md`.

## Pre flight
1. Locate the task folder: `find tasks/ -type d -name "<UUID>*"`.
2. Read `rubrics.md` and parse the list of `(N, result, rationale)`, the overall rating, the overall rationale, the sufficient yes or no, and the missing aspects text.
3. Confirm the Feather tab is on the right task URL via `tabs_context_mcp`.
4. **Dump the existing form state first.** The form may already have stale values from a previous attempt. Read every textarea by id, every Pass/Fail aria-pressed, the rating buttons aria-pressed, and the sufficient buttons aria-pressed. Treat anything already set as suspect, not as truth.

## Inject the textarea helper (once per session)
```javascript
window.__setTextarea = function(textarea, text) {
  const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
  setter.call(textarea, text);
  textarea.dispatchEvent(new Event('input', { bubbles: true }));
  textarea.dispatchEvent(new Event('change', { bubbles: true }));
  return 'ok ' + text.length;
};
'helper injected'
```

## Locate Pass/Fail buttons
The Feather form uses MUI ToggleButtonGroup. Each rubric has its own pair of `<button>Pass</button>` / `<button>Fail</button>`. There are exactly N pairs total (N = number of rubrics) and they appear in DOM order matching the rubric order. Climbing up the DOM from the rationale `<textarea>` does NOT reach the buttons because they live in a separate subtree of the form layout. Use the global ordered list instead:

```javascript
const passButtons = [...document.querySelectorAll('button')].filter(b => b.textContent.trim() === 'Pass');
const failButtons = [...document.querySelectorAll('button')].filter(b => b.textContent.trim() === 'Fail');
// passButtons[i] / failButtons[i] correspond to rubric (i + 1) in textarea id order
```

Determine textarea id order by reading the rubric textarea ids in DOM order (e.g. `root_rubrics_visual-1_rationale`, `root_rubrics_content-2_rationale`, ...). The Pass/Fail index aligns with this order, NOT with the numeric suffix in the id.

## For each rubric N
1. **Set the rationale textarea by id** (textarea ids are stable: `root_rubrics_<group>-<n>_rationale`):
   ```javascript
   const t = document.getElementById('root_rubrics_<group>-<n>_rationale');
   window.__setTextarea(t, '<rationale>');
   ```
   Never use `form_input` or character typing on textareas. They time out and garble long text.

2. **Click Pass or Fail only if not already pressed.** MUI toggles flip on every click, so clicking an already pressed Pass will deselect it. Always check `aria-pressed === 'true'` first:
   ```javascript
   const target = wantPass ? passButtons[i] : failButtons[i];
   if (target.getAttribute('aria-pressed') !== 'true') target.click();
   ```
   If `target.click()` fails to register (some MUI states need a real pointer event), fall back to `find` + `computer.left_click` with the returned ref.

## Overall Assessment
1. Click overall rating 1, 2, or 3 only if not already pressed:
   ```javascript
   const ratingBtns = [...document.querySelectorAll('button')]
     .filter(b => ['1','2','3'].includes(b.textContent.trim()));
   const target = ratingBtns.find(b => b.textContent.trim() === '<RATING>');
   if (target.getAttribute('aria-pressed') !== 'true') target.click();
   ```
2. Set the overall rationale textarea by id:
   ```javascript
   window.__setTextarea(document.getElementById('root_overall_assessment_overall_rationale'), '<text>');
   ```
3. Set the sufficient toggle. The two buttons:
   - "Yes, the rubrics were sufficient"
   - "No, important aspects were missing"
   
   `button.click()` from JS sometimes only deselects the current option without selecting the other in MUI exclusive groups. If after clicking, both report `aria-pressed === 'false'`, fall back to `find` + `computer.left_click`.
4. **Always handle the missing field consistently with the sufficient choice:**
   - If sufficient = Yes → clear `root_overall_assessment_rubrics_missing` (`__setTextarea(t, '')`). Do not leave stale text from a previous attempt.
   - If sufficient = No → set `root_overall_assessment_rubrics_missing` with the missing aspects text from `rubrics.md`.

## Verify (mandatory before reporting ready)
Dump the final state and confirm every field. Use `console.log` + `read_console_messages` for long values to bypass the safety filter on raw return values:

```javascript
(() => {
  const rubricIds = [/* in DOM order */];
  const passed = [...document.querySelectorAll('button')]
    .filter(b => b.textContent.trim() === 'Pass')
    .map(b => b.getAttribute('aria-pressed'));
  const rating = [...document.querySelectorAll('button')]
    .filter(b => ['1','2','3'].includes(b.textContent.trim()))
    .map(b => b.textContent.trim() + '=' + b.getAttribute('aria-pressed'));
  const sufficient = [...document.querySelectorAll('button')]
    .filter(b => /Yes, the rubrics|No, important/.test(b.textContent.trim()))
    .map(b => b.textContent.trim().slice(0,3) + ':' + b.getAttribute('aria-pressed'));
  const lens = rubricIds.map(id => id + '=' + document.getElementById(id).value.length);
  const overall = document.getElementById('root_overall_assessment_overall_rationale').value.length;
  const missing = document.getElementById('root_overall_assessment_rubrics_missing').value.length;
  console.log('FIN', JSON.stringify({passed, rating, sufficient, lens, overall, missing}));
})()
```

Then `read_console_messages` with pattern `FIN`. Confirm:
- All wanted Pass/Fail buttons report `aria-pressed=true`, exactly one per rubric
- Exactly one rating button is pressed and matches the intended rating
- Exactly one sufficient button is pressed and matches the intended choice
- Every rationale length is > 0 and matches what `rubrics.md` says
- Missing field length is 0 if sufficient=Yes, > 0 if sufficient=No

If any check fails, fix it before reporting ready.

## Cross field consistency rules (apply BEFORE filling and AGAIN before reporting ready)
1. **Pass/Fail counts vs rating:**
   - All rubrics Pass → rating 3, unless a polish/aesthetic failure not covered by any rubric justifies dropping to 2 (mention it explicitly in the overall rationale)
   - 1 or 2 minor Fails → rating 2
   - Critical Fails (app does not render, core feature broken) → rating 1
2. **Overall rationale must explicitly justify the chosen rating.** If rating is 2 with all rubrics Pass, the rationale MUST state the specific reason (which polish or UX issue, where) so the rating does not look arbitrary.
3. **Sufficient vs missing field:**
   - sufficient=Yes → missing field MUST be empty. Never leave stale text.
   - sufficient=No → missing field MUST be populated, AND the overall rationale should acknowledge that the missing concern is NOT being used to lower the rating (otherwise it looks like you double counted)
4. **Each rubric rationale must align with its Pass/Fail.** Do not write a rationale that describes a failure on a rubric marked Pass without explicitly noting why it still passes the literal criterion (and pointing to the overall rationale for the consequence).
5. **No two fields may contradict each other.** If you mention a defect in any rationale, the overall rationale must reflect it consistently.

Run this checklist twice: once after parsing `rubrics.md` (warn the user if it is internally inconsistent), and once after filling the form (verify the live state matches).

## Final report
Take a screenshot. Print:
```
FORM FILLED, PLEASE REVIEW AND SUBMIT MANUALLY
Do not click the "In progress" dropdown or "Mark as complete". User only.
```
