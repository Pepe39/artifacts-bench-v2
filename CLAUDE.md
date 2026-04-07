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
- Textareas: use the React native setter helper (`window.__setTextarea`). Never use `form_input` or character typing on textareas.
- Overall rating must match the Pass and Fail counts. All pass goes to 3, a couple of minor fails goes to 2, critical fails go to 1.
- Rationales must be specific. Say what was tested and what happened.

## Task type
This project handles single artifact rubric evaluation. The `ui-berry` project handles Pairwise Preference. Do not mix them.
