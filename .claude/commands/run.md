# Command: run

## Purpose
Open a Feather Rubric Rating task by UUID and run the full evaluation: extract the prompt, source, and rubrics, set up the artifact locally, test it in Brave, fill the form. Never submits.

## Usage
```
/run <UUID>
```
Example: `/run 65c1270d-cbae-40e5-b880-21c3d0e10824`

The pipeline navigates to `https://msft.feather-prod.azure.com/tasks/<UUID>` and runs through to "FORM FILLED, PLEASE SUBMIT MANUALLY".

If `tasks/*/<UUID>/progress.md` already exists, resume from `CURRENT_STEP`.

## Critical rule
Never click the "In progress" dropdown or "Mark as complete". The user submits.

## Prerequisites
- Claude Code launched with `--chrome` (Brave plus the claude in chrome extension)
- Logged in to Feather
- `steps.md` open as the workflow reference

## Task folder
```
tasks/YYYY-MM-DD/<UUID>/
├── task_info.md     URL, UUID, type, batch, date
├── progress.md      CURRENT_STEP (claimed | extracted | evaluated | filled | confirmed)
├── prompt.md        Prompt extracted from Feather
├── source.tsx       Source code from the Source Code tab
├── rubrics.md       Each rubric with Pass or Fail, rationale, overall
├── screenshots/     Local preview and interaction evidence
└── local/           Local Vite project (always created)
```

## Phases

### Phase 0: setup
1. Validate `$ARGUMENTS` matches a UUID. If not, error out.
2. `find tasks/ -type d -name "<UUID>*"`. If found, read `progress.md` and resume from the matching phase.
3. `mcp__claude-in-chrome__tabs_context_mcp`. If it fails, ask the user to launch with `--chrome`.

### Phase 1: open the task
1. `tabs_create_mcp` to create a new tab.
2. `navigate` to `https://msft.feather-prod.azure.com/tasks/<UUID>`.
3. Wait 4 to 5 seconds for the page to settle.
4. `get_page_text` and confirm the title contains "Rubric Rating". If not, abort with the actual title.
5. Create `tasks/<DATE>/<UUID>/` (use `date +%Y-%m-%d`).
6. Write `task_info.md` with UUID, URL, task type from the title, batch, date.
7. Write `progress.md` as `CURRENT_STEP: claimed`.

### Phase 2: extract
Run the logic in `.claude/commands/extract.md`:
1. Save the prompt to `prompt.md`.
2. Extract source via the iframe served URL technique documented in `extract.md` (navigate to `<iframe>/src/App.tsx`, `window.__src = document.body.innerText`, `console.log('SRCDUMP', __src)`, `read_console_messages` pattern `SRCDUMP`). Do NOT rely on Monaco APIs, `.view-line`, hidden textarea, or clipboard. They all fail.
3. Save to `source.tsx`.
4. Parse the rubric list and write a skeleton `rubrics.md`.
5. `progress.md` → `CURRENT_STEP: extracted`.

### Phase 3: local setup and testing (mandatory)
Always run the artifact locally inside the task folder. The Feather Live Preview is only a sanity check.

1. Create the local project:
   ```bash
   cd "tasks/<DATE>/<UUID>" && \
   mkdir -p local && cd local && \
   npm create vite@latest . -- --template react-ts -y && \
   npm install
   ```
2. Replace `local/src/App.tsx` with the contents of `../source.tsx`.
3. Detect the export name in `source.tsx` (for example `export default function GardenPlanner`) and patch `local/src/main.tsx` so the import matches:
   ```tsx
   import React from 'react'
   import ReactDOM from 'react-dom/client'
   import App from './App'
   import './index.css'
   ReactDOM.createRoot(document.getElementById('root')!).render(
     <React.StrictMode><App /></React.StrictMode>
   )
   ```
4. If `source.tsx` imports anything beyond `react` and `react-dom` (lucide, framer motion, tailwind, etc), install those deps. For tailwind, run the standard `npx tailwindcss init -p` setup and add the directives to `src/index.css`.
5. Start the dev server in the background:
   ```bash
   cd "tasks/<DATE>/<UUID>/local" && npm run dev
   ```
   Use `run_in_background: true`. Wait around 3 seconds, read the background output, and capture the actual port (Vite shifts to 5174+ if 5173 is taken).
6. Open the local URL in Brave with `tabs_create_mcp` and `navigate http://localhost:<PORT>`. Take a screenshot.
7. `read_console_messages` to catch runtime errors. If there are errors, fix them or treat them as fail evidence.
8. For each rubric, interact with the local app in Brave (click buttons, fill forms, drag, edge cases), save screenshots into `screenshots/`, decide Pass or Fail with a concrete rationale (what you tested, what happened).
9. Read `source.tsx` and verify state management, event handlers, types, no direct mutations, edge case handling.
10. Optionally cross check against the Feather Live Preview tab.
11. Update `rubrics.md` with results, overall rating, overall rationale, and the "rubrics sufficient" answer.
12. **Run the cross field consistency checklist from `fill.md` against `rubrics.md` BEFORE moving on.** Specifically:
    - Pass/Fail counts must justify the rating. All Pass + rating < 3 requires explicit polish/UX reason in the overall rationale.
    - Each rubric rationale must align with its Pass/Fail. If you describe a defect on a Pass rubric, explicitly note why it still passes the literal criterion and defer the consequence to the overall rationale.
    - sufficient=Yes → missing field empty. sufficient=No → missing field populated AND overall rationale must state the missing concern is NOT being used to lower the rating.
    - No two rationales may contradict each other. Rationale lengths should be roughly comparable (no rubric with a 1-line rationale next to 5-line ones).
    If any check fails, fix `rubrics.md` before continuing.
13. Stop the dev server when done.
14. `progress.md` → `CURRENT_STEP: evaluated`.

### Phase 4: fill the form
Run the logic in `.claude/commands/fill.md` end to end. Do not skip its pre-flight dump, its global Pass/Fail targeting (never climb up from textareas), the aria-pressed check before clicking MUI toggles, or the mandatory verify dump via `console.log` + `read_console_messages`. Re-run the cross field consistency checklist against the live form state before reporting ready.
1. Inject the `window.__setTextarea` helper.
2. For each rubric N, click Pass or Fail (only if not already pressed) and set the rationale via the helper, by textarea id.
3. Click overall rating 1, 2, or 3 (only if not already pressed) and set the overall rationale.
4. Click "Yes" or "No" sufficient. ALWAYS clear the missing field if Yes; ALWAYS set it if No.
5. Verify dump and confirm every field matches `rubrics.md`. Fix any discrepancy before reporting.
6. `progress.md` → `CURRENT_STEP: filled`.
7. Stop and print:
   ```
   FORM FILLED, PLEASE REVIEW AND SUBMIT MANUALLY
   ```

### Phase 5: confirm
After the user says `listo`: `progress.md` → `CURRENT_STEP: confirmed`, `TASK_STATUS: complete`.

## Resume map
```
claimed   → Phase 2
extracted → Phase 3
evaluated → Phase 4
filled    → Phase 5
confirmed → done
```

## References
- `steps.md`: full workflow
- `tasks/2026-04-06/65c1270d-cbae-40e5-b880-21c3d0e10824/`: reference completed task
