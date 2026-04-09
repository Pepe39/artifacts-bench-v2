# Command: review

## Purpose
Open a Feather review task by UUID, do the same end-to-end work an attempter would do (download source, run locally, test every rubric), then compare against the attempter's submission and produce a friendly natural-language verdict plus a 1-to-5 annotation score. Never submits.

## Usage
```
/review <UUID>
/review <N>
```
Examples:
- `/review d483a26e-e0a7-4be2-a8cd-d8402bb5f0b6` runs one review for that UUID.
- `/review 5` starts batch mode for 5 consecutive reviews. Ask the user for the first UUID, run the full pipeline end to end (Phases 0 to 5, including writing `feedback_to_attempter.md`), then stop and ask for the next UUID. Repeat until N reviews are done. Do not wait for `listo` between items in a batch, Phase 6 confirmation is skipped inside the batch and only the final item waits for the user. Keep a running counter in the verdict header like `Batch 2/5`. If the user passes a UUID after the batch number (`/review 5 <UUID>`), use it as the first one without asking.

## Time budget (target 15 minutes)
| Phase | Budget | Cumulative |
|---|---|---|
| 1. Open task and snapshot attempter work | 2 min | 2 min |
| 2. Run the artifact end to end like /run, decide your own Pass/Fail | 6 min | 8 min |
| 3. Compare your decisions against the attempter, list discrepancies | 3 min | 11 min |
| 4. Score 1 to 5, decide rework vs apply vs accept | 2 min | 13 min |
| 5. Report friendly verdict | 2 min | 15 min |

If the artifact is so broken or the attempter work is so bad that Phase 2 will clearly blow past 6 minutes, short circuit to score 1 or 2 and move on.

## Hard rules
- Never click the "In progress" dropdown or "Mark as complete". The user submits.
- Never modify the extracted `source.tsx`. Bugs are findings, not things to fix.
- Wording rule for any rationale you write or rewrite: never use em dashes, never use hyphens as sentence connectors, avoid sentences structured around `;` or `:`. Plain short sentences with periods only. Parenthetical clarifications in `(parentheses)` are fine.
- Pass rationale phrasing: the canonical Pass rationale is `The candidate meets the requirements for this rubric.` If the attempter wrote `N/A` on a Pass that you also agree is Pass, replace it with that exact sentence and treat it as a minor correction. Do NOT lower the annotation score for this alone.
- All rationales AND all review/feedback files must be in English. No exceptions.
- Always write `feedback_to_attempter.md` at the end of Phase 5 with the 1-5 score, a brief friendly paragraph in past tense (the corrections were already applied or sent back, so describe the mistakes as past), and at least one sentence thanking the attempter for their strong work.
- **Your independent evaluation is NOT the sole source of truth.** The attempter's logic is a second opinion that can be correct. Always do an explicit recheck of your Pass/Fail decisions and rationales against the attempter's before finalizing. When the attempter cites a specific code path, bug, or behavior, verify that claim in the running artifact by dispatching the actual event, clicking the actual button, or inspecting the actual DOM, not by source-reading alone. If the app-tested evidence backs the attempter and contradicts your first read, update YOUR evaluation, not theirs. Credit the attempter when they are right. Be critical of their logic AND of yours.
- **All required local review files must exist before reporting a verdict.** Minimum set: `task_info.md`, `progress.md`, `prompt.md`, `source.tsx` (or extracted HTML), `attempter.md` (verbatim snapshot of the attempter's current form state, one section per rubric plus overall and sufficient), `review.md`, `feedback_to_attempter.md`. Skipping `attempter.md` is a hard failure because without it the comparison in Phase 3 has no recorded baseline and the verdict becomes unauditable.

## Review folder
Reviews live under `reviews/`, not `tasks/`. Same date and UUID layout.
```
reviews/YYYY-MM-DD/<UUID>/
├── task_info.md       URL, UUID, type=review, batch, date
├── progress.md        CURRENT_STEP (claimed | extracted | tested | compared | scored | reported | confirmed)
├── prompt.md          Prompt extracted from Feather
├── source.tsx         Source code from the Source Code tab
├── attempter.md       Attempter snapshot (Pass/Fail, rationales, overall, sufficient)
├── rubrics.md         YOUR independent evaluation, exactly as if running /run
├── review.md          Comparison, corrections, score, friendly verdict
├── feedback_to_attempter.md  Score 1-5 + short friendly paragraph in past tense thanking them and naming the corrected mistakes
├── screenshots/       Local interaction evidence
└── local/             Local Vite project
```

## Phases

### Phase 0: setup
1. Validate `$ARGUMENTS` looks like a UUID. If not, error out.
2. `find reviews/ tasks/ -type d -name "<UUID>*"`. If found, read `progress.md` and resume from the matching phase. A prior `/run` folder for the same UUID is fine, you just add the review files alongside.
3. `mcp__claude-in-chrome__tabs_context_mcp`. If it fails, ask the user to launch with `--chrome`.

### Phase 1: open task and snapshot the attempter (2 min)
1. `tabs_create_mcp` if needed, then `navigate` to `https://msft.feather-prod.azure.com/tasks/<UUID>`.
2. Wait for the page to settle, `get_page_text`, confirm the title contains `Rubric Rating`. Abort with the actual title if not.
3. Create `reviews/<DATE>/<UUID>/`. Write `task_info.md` and `progress.md` (`CURRENT_STEP: claimed`). Save the prompt to `prompt.md`.
4. Snapshot the attempter's existing form state into `attempter.md`. For each rubric, capture the textarea id, the Pass or Fail aria-pressed state, and the rationale text verbatim. Capture overall rating, overall rationale, sufficient answer, and missing field. Format:
   ```
   # Attempter submission (snapshot)
   - Overall rating: <1|2|3>
   - Sufficient: <Yes|No>
   - Missing: <text or empty>

   ## Rubric 1 (root_rubrics_visual-1_rationale)
   - Decision: Pass | Fail
   - Rationale: <verbatim>
   ```
5. Note any immediate red flags into `review.md` under `## First impression`:
   - All Pass with no detail
   - Identical or near-identical rationales across rubrics
   - Blank fields
   - Rationales that do not reference the artifact specifically
   - `N/A` Pass rationales (correction, not penalty)

### Phase 2: run the artifact end to end like /run (6 min)
This is the same flow as `.claude/commands/run.md` Phase 2 + Phase 3, except you write your independent decisions into `rubrics.md` and you do NOT touch the Feather form yet.

1. **Extract source** using the iframe served URL technique from `extract.md`. Find the iframe `src`, navigate to `<iframe>/src/App.tsx` in a separate tab, dump `document.body.innerText` via `console.log('SRCDUMP', __src)`, read with `read_console_messages` pattern `SRCDUMP`. Save to `source.tsx`.
2. **Local setup.** Create the Vite project under `local/`, copy `source.tsx` into `local/src/App.tsx`, patch `local/src/main.tsx` to remove StrictMode and import the detected default export, install any extra deps the source pulls in (lucide, framer motion, tailwind, etc).
3. **Start the dev server** in the background. Read the actual port from the output (Vite shifts to 5174+ if 5173 is taken).
4. **Open in Brave** and `read_console_messages` to catch runtime errors. TDZ, ReferenceError, TypeError or any uncaught exception text is first-class evidence and often invalidates Pass decisions wholesale. Save the raw error text into `rubrics.md` under a `## Console errors` section.
5. **Test every rubric independently.** Click the actual buttons. Drag on the canvas. Move the sliders. Trigger the export. Inspect pixels with `getImageData` if needed for color or render claims. For each rubric write your own Pass or Fail with a 2 to 4 sentence rationale that references specific evidence (which button, what coordinate, what pixel value, what console line).
6. **Read `source.tsx`** to spot-check anything related to logic, randomization, RNG, timing, or client-side constraints that the live test cannot prove on its own.
7. **Write `rubrics.md`** with your independent evaluation in the same shape `/run` produces. Include the sufficient walk and the cross field consistency check from `run.md` Phase 3 step 12.
8. Stop the dev server.
9. `progress.md` → `CURRENT_STEP: tested`.

### Phase 3: compare your decisions against the attempter (3 min)
0. **Runtime recheck of attempter claims.** Before classifying anything, walk the attempter's rationales one by one. For each rationale that makes a specific claim about a code path, bug, or behavior, verify the claim in the running artifact (dispatch the actual DOM event, click the actual button, inspect the actual state). If the attempter says "parseInt('bank') returns NaN and the drop is rejected", simulate the drop and confirm the cell stays empty. If the app-tested evidence backs the attempter and contradicts your independent first read, update YOUR evaluation in `rubrics.md` before moving on. Your evaluation is not the sole source of truth.
1. For each rubric, classify the comparison as one of:
   - **Match** (your decision and rationale agree with the attempter, no rewording needed)
   - **N/A replacement only** (you both Pass, the attempter wrote `N/A`, you would replace it with the canonical Pass phrasing, this is a correction but not a scoring penalty)
   - **Wording-only** (decision is the same but the attempter's rationale is weak, missing evidence, hallucinated, or wrong about a detail)
   - **Flip** (the attempter's Pass or Fail decision is wrong, you would mark it the other way)
2. Compare overall rating. Note if it should change.
3. Compare sufficient answer. Note if it should change. Check that missing field is consistent with the sufficient answer.
4. Write the comparison into `review.md` under `## Comparison`. One section per rubric showing the attempter decision, your decision, and the classification. Also list rating delta and sufficient delta.
5. `progress.md` → `CURRENT_STEP: compared`.

### Phase 4: score 1 to 5 and decide the action (2 min)
Score the attempter's submission on the 1 to 5 scale below. Use the most pessimistic bucket that fits.

| Score | Label | When to use | Action |
|---|---|---|---|
| 5 | Excellent | Zero corrections needed. Every rubric accurate. Rationales are evidence-based. Overall rating well justified. | Accept as is. No rework. |
| 4 | Minor issues | Mostly correct. Only small fixes needed. One rubric flip OR a small rewording OR an `N/A` replacement on a correctly-Passed rubric. | Accept as is. No rework. |
| 3 | Borderline | Engaged with the task but has noticeable issues. Could be one significant flip plus weak rationales, or several wording fixes, or an inconsistent rating vs Pass/Fail counts. | Stop and ask the user whether to apply the corrections to the form or send back for rework. |
| 2 | Excessive LLM misuse | Clearly copy-pasted LLM output. Generic rationales. Hallucinated observations. Does not reference the specific artifact. | Send back for rework. Do not apply corrections. |
| 1 | Spam / Blank | Blank fields, placeholder text, completely unusable output. No genuine engagement. | Send back for rework. Do not apply corrections. |

Hard scoring principles:
- An `N/A` replacement on a correctly-Passed rubric is a correction but is NOT a scoring penalty by itself. A submission that is otherwise perfect but uses `N/A` lands at 4, never lower for that reason alone, and never at 5 because zero-corrections is the bar for 5.
- Two or more rubric flips or one flip plus several rationale rewrites lands at 3.
- Hallucinated features that do not exist in the artifact, or claims that contradict the source, lands at 2.
- Blank, placeholder, or unusable submissions land at 1.

Write the score and the action into `review.md` under `## Annotation score`. Justify the score in 2 to 4 sentences listing the specific corrections that drove it.

`progress.md` → `CURRENT_STEP: scored`.

### Phase 5: report friendly verdict (2 min)
Print a short, friendly, direct verdict to the user in this shape:

```
REVIEW VERDICT — <UUID>
Score: <1-5> (<label>)

Hey, here is what I found:
<2 to 4 sentences in friendly natural language. Speak directly to the issues. No corporate tone, no jargon, no em dashes. If the attempter did good work, say so. If they missed things, name them specifically (which rubric, what the actual behavior is). Mention the rating delta and sufficient delta if any. End with the recommended action.>

Action: <accept | apply corrections | send back for rework>

Top findings:
- <finding 1>
- <finding 2>
- <finding 3>

Full corrected rubric set in reviews/<DATE>/<UUID>/review.md.
```

Action mapping:
- Score 5 or 4 → `Action: accept`. Do not apply corrections, do not send back. The user submits as is.
- Score 3 → `Action: ask user`. Stop after the verdict and ask the user explicitly: `This is borderline. Want me to apply the corrections to the form, or send it back for rework?` Wait for the user's call before doing anything else.
- Score 2 or 1 → `Action: send back for rework`. Do not apply corrections.

Then print the standard reminder:
```
Do not click "In progress" or "Mark as complete". User decides next step.
```

Then write `feedback_to_attempter.md` in the review folder. It must contain:
- A header line with the score in `**Score: N / 5 (label)**` format
- One short paragraph (4 to 8 sentences) in English, in past tense (the corrections were already applied or sent back, so describe the mistakes as past), naming the specific rubrics that were wrong and what the actual behavior was
- At least one sentence thanking the attempter for their strong work
- Wording rule applies (no em dashes, no hyphens as connectors, no `;`/`:` sentence structures, plain short sentences)

`progress.md` → `CURRENT_STEP: reported`.

### Phase 6: confirm
After the user says `listo`: `progress.md` → `CURRENT_STEP: confirmed`, `TASK_STATUS: complete`.

## Resume map
```
claimed   → Phase 1 (rest of)
extracted → Phase 2
tested    → Phase 3
compared  → Phase 4
scored    → Phase 5
reported  → Phase 6
confirmed → done
```

## Applying corrections to the form

### Auto-apply on score 4 or 5
If the score is 4 or 5 AND the only corrections are `N/A` Pass rationale replacements (no flips, no rationale rewrites, no rating change, no sufficient change), automatically apply those replacements to the form during Phase 5, before printing the verdict. Then mention in the verdict that the N/A replacements were already applied. This is the default behavior, no user confirmation needed for the N/A swap because it is a mechanical correction with no scoring impact.

If the score is 4 or 5 but corrections include anything beyond N/A swaps, do NOT auto-apply. Print the verdict with `Action: accept` and let the user decide.

### Score 3 or explicit request
This command produces a verdict. For non-trivial corrections it does NOT write back to the Feather form by default. Only apply non-trivial corrections when:
- Score is 3 AND the user explicitly says to apply, OR
- The user asks you to apply for any score

When applying, use the same fill mechanics from `.claude/commands/fill.md`:
- `window.__setTextarea` helper for textareas
- Global Pass/Fail targeting via filtered button arrays, never climb up from textareas
- aria-pressed check before clicking MUI toggles
- Fall back to `find` + `computer.left_click` when `.click()` does not register on MUI ToggleButtonGroup
- Verify dump via `console.log('FIN', ...)` + `read_console_messages` pattern `FIN` before reporting ready
- Re-run the cross field consistency checklist from `fill.md` against the live form state
- For any Pass rubric where the attempter wrote `N/A`, set the rationale to `The candidate meets the requirements for this rubric.`

After applying, print:
```
CORRECTIONS APPLIED, PLEASE REVIEW AND SUBMIT MANUALLY
Do not click "In progress" or "Mark as complete". User submits.
```

## Friendly tone reminder
The verdict is the part the user actually reads. Keep it short, plain, and direct. Examples:

Good (score 5):
```
Hey, this one is clean. All 14 rubrics line up with what I tested locally, the rationales reference real behavior, and the rating matches the Pass/Fail counts. Nothing to fix.
Action: accept
```

Good (score 4 with N/A replacement):
```
Solid work. The decisions are all correct and the rationales are specific. Only catch is that rubrics 2 and 7 use the old "N/A" Pass phrasing. I would swap those for the canonical Pass sentence, but no scoring penalty for it.
Action: accept
```

Good (score 3 with one flip):
```
Mostly there but rubric 8 is marked Pass when the regenerate button is actually inert. The attempter probably did not click it. The rest checks out and the rating still feels right at 2. Borderline call on whether to fix and submit or bounce back.
Action: ask user
```

Good (score 2):
```
This reads like a copy-paste. Rubrics 4, 7, 9, 11 all have the same generic rationale and rubric 6 claims an export feature that does not exist in the source. I would not patch this, send it back.
Action: send back for rework
```

Good (score 1):
```
Half the rationale fields are blank and the overall rationale is one line. Send it back.
Action: send back for rework
```

Bad (do not write like this):
```
The submission demonstrates a moderate level of engagement; however, several criteria warrant additional scrutiny — specifically rubrics 5, 7, and 9 — which collectively suggest the attempter may have insufficiently exercised the interactive elements.
```

## References
- `Reviewer_Workflow.docx`: source of truth for the reviewer process
- `.claude/commands/run.md`: attempter pipeline this command mirrors in Phase 2
- `.claude/commands/extract.md`: source extraction technique
- `.claude/commands/fill.md`: form fill mechanics for MUI toggle groups and textareas
- `CLAUDE.md`: hard rules and wording rule
