# artifact-bench-embed-html-issues

Project for evaluating Feather Rubric Rating tasks (`weighted_aesthetics_*`). Each task is a single artifact scored against N rubrics (Pass or Fail) plus a 1 to 3 overall rating.

## Workflow
- Full guide: `steps.md`
- Commands: `.claude/commands/run.md`, `extract.md`, `fill.md`
- Reference task: `tasks/2026-04-06/65c1270d-cbae-40e5-b880-21c3d0e10824/`

## Hard rules
- Never click the "In progress" dropdown or "Mark as complete". The user submits.
- Launch Claude Code with `--chrome` (Brave plus the claude in chrome extension).
- Always pull `source.tsx` from the Source Code tab and run it locally inside `tasks/<DATE>/<UUID>/local/` (Vite, React, TypeScript). Test in Brave at `http://localhost:5173`. Do not trust the Feather Live Preview alone.
- **Never modify the extracted `source.tsx`.** It is a verbatim copy of the artifact under evaluation and must be used as-is in `local/src/App.tsx`. Bugs (TDZ errors, type errors, runtime crashes, missing deps, broken logic) are findings to report in `rubrics.md` — never fix, patch, or "clean up" the source. Editing it invalidates the evaluation and produces inconsistent results across runs/teammates.
- Textareas: use the React native setter helper (`window.__setTextarea`). Never use `form_input` or character typing on textareas.
- Overall rating must match the Pass and Fail counts. All pass goes to 3, a couple of minor fails goes to 2, critical fails go to 1.
- Rationales must be specific. Say what was tested and what happened.
- Wording rule for all rationales (rubric, overall, missing) AND all review/feedback files: always in English. Never use em dashes (—) or hyphens as sentence connectors, and avoid sentences structured around `;` or `:`. Use plain short sentences with periods. Parenthetical clarifications in `(parentheses)` are fine.
- **Always double-verify in BOTH the Feather live preview AND a local run before scoring or reviewing.** Reading the source code alone is never enough, even for tiny HTML artifacts. Click the actual buttons, observe the rendered DOM after interaction, confirm visual state changes. Source-only review misses runtime bugs and visual regressions. After verification is complete and the task is reported, clean up any test artifacts (local Vite folders, screenshots used only for verification) that are not needed for the final deliverable.
- **Final consistency double-check MUST be done on the Feather platform itself, not from memory, local files, or the plan.** Before reporting a fill (attempt) or a review as ready, read each rubric question text directly from the live DOM in the Feather tab and compare it against the textarea value you just wrote for that rubric. Verify rubric question ↔ decision ↔ rationale alignment on the actual form. This catches swapped rationales, wrong Pass/Fail toggles that silently failed (MUI ToggleButtonGroup issue), and stale state from previous runs. Applies equally to `/run` (attempts) and `/review` (corrections applied to the form).
- **For `/review`, your independent evaluation is NOT the sole source of truth.** The attempter's logic can be correct and must be credited when it is backed by evidence from the tested application. Always perform an explicit recheck of your evaluation against the attempter's before finalizing a verdict. Be critical of the attempter's logic, but equally critical of your own. When the attempter cites a specific code path, bug, or behavior, verify that claim in the running artifact (dispatch the event, click the button, inspect the DOM), not only by source-reading. If the app-tested evidence backs the attempter and contradicts your initial read, update your evaluation, not theirs.
- **For `/review`, ALL required local files must be created in `reviews/<DATE>/<UUID>/` before reporting a verdict.** Minimum set: `task_info.md`, `progress.md`, `prompt.md`, `source.tsx` (or the extracted HTML), `attempter.md` (verbatim snapshot of the attempter's current form state), `review.md` (your independent evaluation plus comparison, score, and action), `feedback_to_attempter.md`. Missing the `attempter.md` snapshot is a hard failure because without it there is no recorded baseline to compare against.

## Task type
This project handles single artifact rubric evaluation. The `ui-berry` project handles Pairwise Preference. Do not mix them.
