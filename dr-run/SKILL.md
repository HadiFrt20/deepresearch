---
name: dr-run
description: Execute the next batch of research tasks autonomously. Reads todo.md, spawns subagents for each task, verifies output, and loops until the current phase is complete or a stop condition is hit. Use after /dr-new has scaffolded the project.
---

You are executing an autonomous research loop. Read CLAUDE.md for the full execution protocol.

Accepts optional arguments: `/dr-run`, `/dr-run sequential`, `/dr-run parallel`, `/dr-run parallel-auto-improve`, `/dr-run sequential-auto-improve`, `/dr-run auto`, `/dr-run parallel auto`, `/dr-run sequential auto`, `/dr-run --no-adversary`, `/dr-run parallel --no-adversary`.

## Auto mode

/dr-run supports an `auto` flag that suppresses all permission prompts and user interruptions.

Usage:
  /dr-run              → default, asks for approval on sensitive operations
  /dr-run auto         → fully autonomous, no prompts
  /dr-run parallel auto → combine with parallel mode
  /dr-run sequential auto → explicit sequential + auto

When auto mode is active:
- All web searches, fetches, and scrapes proceed without asking
- All file writes proceed without asking
- All bash commands needed for research proceed without asking
- Subagents spawn with inherited permissions (no extra approval per subagent)
- User is NOT interrupted between tasks
- Research runs uninterrupted until: phase completes, stop condition hits, or auto mode is explicitly disabled

Safety in auto mode:
- The researcher subagent still follows all rules in CLAUDE.md (NOT_FOUND over guessing, source URLs required, 10-min time budget per task)
- Rate-limit detection still triggers checkpoints (see CHANGELOG)
- 3 consecutive failures still stops the run
- 3+ hour cumulative runtime still stops the run
- User can interrupt at any time with Ctrl+C

Auto mode is the recommended setting for overnight runs and long research sessions.

## Execution Loop

### Step 0: Determine Execution Mode, Permission Mode, and Adversary Flag

1. Check if the user passed a mode argument: `sequential` | `parallel` | `sequential-auto-improve` | `parallel-auto-improve`
2. Check if the user passed `auto` or `ask` as an additional argument
3. Check if the user passed `--no-adversary` flag
4. If no mode argument, read `CLAUDE.md` for the "Execution Mode" section and use the default mode listed there
5. If no permission argument, read `CLAUDE.md` for the "Permission Mode" section. If absent, default to `ask`
6. If `CLAUDE.md` has no "Execution Mode" section, fall back to `sequential`

Print: `Mode: {mode} | Permissions: {auto or ask} | Adversary: {enabled or disabled}`

When `--no-adversary` is passed:
- All adversary batches in Step 10.5 are skipped
- Provenance envelopes are still produced by the researcher (`adversary_verdict` stays `"pending"` on all claims)
- No sidecar files are created

### Step 0.5: Dependency Analysis (parallel modes only)

If mode is `parallel` or `parallel-auto-improve`:

1. Check if `.research/parallel-plan.md` exists AND was created after the last `todo.md` change (compare file modification times using `stat`)
2. If the plan does not exist or is stale:
   a. Spawn the `dr-planner` subagent to analyze the current phase. Use the Agent tool with `subagent_type: "dr-planner"`.
   b. After the planner returns, **STOP** and show the plan summary to the user.
   c. Print: "Parallel plan written to `.research/parallel-plan.md`. Review it, then run `/dr-run parallel` again to proceed."
   d. If the plan recommends **SEQUENTIAL ONLY**, print: "The planner recommends sequential execution for this phase. Consider running `/dr-run sequential` instead."
   e. **Do NOT proceed automatically** — force the user to confirm by re-running the command after reviewing the plan.
3. If the plan exists and is fresh, re-read `.research/parallel-plan.md` and proceed using its recommended strategy.

### Step 1: Find Current Phase

Read `.research/ROADMAP.md`. Find the current phase — the first phase with status `NOT_STARTED` or `IN_PROGRESS`. If all phases are `COMPLETE`, tell the user and suggest `/dr-report`.

### Step 2: Activate Phase

Update the current phase status to `IN_PROGRESS` in `.research/ROADMAP.md`.

### Step 3: Find Next Task(s)

Read `todo.md`.

**Sequential modes:** Find the first unchecked task matching `- [ ] P{current_phase}.*`. If no unchecked tasks remain for this phase, go to Step 11.

**Parallel modes:** Read `.research/parallel-plan.md` for the task classification table. Collect the next batch of up to 5 unchecked tasks that are classified as SAFE (or RISKY with merge strategy). Do not include BLOCKING tasks whose dependencies haven't completed yet. If no unchecked tasks remain for this phase, go to Step 11.

### Step 4: Load Schema

Read `.research/ARCHITECTURE.md` to get the output schema for this phase's entity type.

### Step 5: Spawn Researcher(s)

**Sequential modes:** Spawn ONE `dr-researcher` subagent with this prompt:

```
Your task: {task description from todo.md}
Output schema: {relevant schema from ARCHITECTURE.md}
Write to: {file path specified in the task}
Follow the data rules in CLAUDE.md. You have a 10-minute time budget.
```

If auto mode is active, prepend this instruction to the subagent prompt:
```
Run autonomously. Do not ask for permission on any tool call. Complete the task or report NOT_FOUND for missing data.
```

Use the Agent tool with `subagent_type: "dr-researcher"` and include the full task context.

**Parallel modes:** Spawn up to 5 `dr-researcher` subagents **in a single message** using multiple Agent tool calls. Each subagent gets the same prompt format as above (including the auto mode prefix if active) but for its specific task. If the parallel plan requires merge strategy for RISKY tasks, instruct each subagent to write to `data/{file}.{task_id}.tmp.json` instead of the canonical file.

### Step 6: Verify Output

Check each subagent's output:
- a) File exists at the specified path
- b) File parses as valid JSON or valid markdown (depending on expected format)
- c) Required fields are populated (not empty strings, not null)
- d) `sources` array has at least 1 URL
- e) `status` field is set to COMPLETE, INCOMPLETE, or INACCESSIBLE

### Step 7: Mark Success

If all verification checks pass:
- Change `- [ ]` to `- [x]` in `todo.md` for this task

### Step 8: Mark Failure

If any verification check fails:
- Change `- [ ]` to `- [!]` in `todo.md` for this task
- Note which checks failed

### Step 8.5: Merge Temp Files (parallel modes with merge strategy)

If the parallel plan required temp files for RISKY tasks:
1. After the batch completes, read all `data/{file}.{task_id}.tmp.json` files
2. Merge entries into the canonical output file (append to existing array)
3. Delete the temp files after successful merge
4. If merge fails, log the error and leave temp files in place for manual review

### Step 9: Log to Changelog

Append to `.research/CHANGELOG.md`:
```
[ISO timestamp] | {task_id} | DONE or FAIL | {output file} | {one-line summary} | mode:{sequential or parallel}
```

For parallel batches, log all tasks in the batch together:
```
[ISO timestamp] | BATCH | {task_ids} | mode:parallel | {done_count} done, {fail_count} failed
```

### Step 10: Checkpoint

Every 5 tasks (or after each parallel batch), write a checkpoint entry to `.research/CHANGELOG.md`:
```
[ISO timestamp] | CHECKPOINT | completed: {N}, failed: {N}, remaining: {N} | mode:{mode} | patterns: {observations} | recommendation: continue/stop
```

Check stop conditions:
- All phase tasks checked → go to Step 11
- 3+ hours elapsed → stop with time warning
- 3+ failures in one batch (parallel) → fall back to sequential for the rest of the phase, log: `[ISO timestamp] | MODE_FALLBACK | parallel→sequential | reason: {N} failures in batch`
- 3 consecutive failures (sequential or after fallback) → stop the run with failure warning
- Rate limited 3 times → stop with rate limit warning

**Auto mode behavior:** When auto mode is active, do not print intermediate progress messages between tasks. Just log to CHANGELOG and keep going. Only print output at checkpoints and phase completion.

If not stopped, go to Step 3.

### Step 10.5: Adversary Batch

**Skip this step if `--no-adversary` flag was passed.**

After the checkpoint fires every 5 tasks (or after each parallel batch), run adversarial verification on the completed tasks from this batch:

1. **Collect completed tasks:** Gather the `[x]` tasks from this batch only. Filter out `[!]` (failed) and `[ ]` (unchecked) tasks. If no `[x]` tasks in this batch, skip.

2. **Load provenance fields:** Read `provenance_fields` from `.research/ARCHITECTURE.md`. The adversary only attacks claims on these fields.

3. **Spawn adversary:** Spawn the `dr-adversary` subagent with this prompt:

```
Verify the claims in these completed research entries.

Data files to check: {list of output file paths for the batch's [x] tasks}
Provenance fields to verify: {provenance_fields from ARCHITECTURE.md}
Source cache: .research/source-cache/
Adversary schema: prompts/adversary-schema.json

Write your verdicts to: data/{original-file}.adversary.json

If the sidecar file already exists, read it first and append your new verdicts to the existing array.

You have 5 minutes. Focus on provenance_fields claims only.
```

Use the Agent tool with `subagent_type: "dr-adversary"`.

**Sequential mode:** Adversary blocks (runs synchronously before next research task). ~4% time overhead.
**Parallel mode:** Adversary runs in the background (non-blocking) using `run_in_background: true`. Results merge into sidecar files when ready. Near-zero overhead because researchers and adversary write to different files (researchers → `data/*.json`, adversary → `data/*.adversary.json`).

4. **Validate adversary output:** After the adversary completes, validate its output against `prompts/adversary-schema.json`:
   - If verdict is `refuted` or `weakened` and `adversary_evidence` is empty or missing → downgrade verdict to `unverifiable`
   - If verdict is `weakened` and `confidence_delta` is null → set `confidence_delta` to `-0.3`
   - If output fails schema validation entirely → claims stay `pending`, log warning to CHANGELOG: `[timestamp] | adversary_batch | FAIL | [batch_task_ids] | Adversary output malformed, claims unverified`

5. **Calculate survival scores** for each claim using the adversary verdicts:
   - `confirmed`: `survival_score = confidence_score` (unchanged)
   - `weakened`: `survival_score = max(0.0, confidence_score + confidence_delta)`
   - `refuted`: `survival_score = 0.0`
   - `unverifiable`: `survival_score = confidence_score * 0.5`

6. **Log batch summary** (3 lines to stdout):
```
Adversary batch {N}: {X} claims tested | confirmed: {A}, weakened: {B}, refuted: {C}, unverifiable: {D}
lowest: "{claim_text}" (score: {0.XX})
```

7. **Error handling:**
   - Adversary timeout (>5 min): claims stay `pending`, non-blocking warning in CHANGELOG
   - Adversary crash: claims stay `pending`, mark batch as failed in CHANGELOG, continue to next research task
   - Sidecar file write failure: log warning, adversary results lost for this batch but run continues

### Step 11: Phase Complete

When all tasks for the current phase are checked:
1. Update phase status to `COMPLETE` in `.research/ROADMAP.md`
2. Write phase completion entry to CHANGELOG
3. Count done `[x]` and failed `[!]` tasks for this phase

### Step 11.5: Auto-Improve (auto-improve modes only)

If mode is `sequential-auto-improve` or `parallel-auto-improve`:
1. Before starting the next phase, automatically run the `/dr-improve` loop against the completed phase's data
2. Log to CHANGELOG: `[ISO timestamp] | AUTO_IMPROVE | phase:{N} | old_pass_rate:{X}% | new_pass_rate:{Y}% | decision:{kept or reverted}`
3. If improved: next phase uses the new researcher. If reverted: next phase uses the previous researcher.
4. Continue to the next phase automatically.

### Step 12: Completion Output

Print results based on mode:

**Sequential modes:**
```
Phase {N} complete. {done} done, {failed} failed.

Next steps:
  /dr-status   — see overall progress
  /dr-review   — validate data quality for this phase
  /dr-improve  — run self-improvement loop on the researcher
  /dr-run      — continue to next phase
```

**Parallel modes:** Also show batch-level stats:
```
Phase {N} complete. {done} done, {failed} failed.
Batches: {batch_count} | Fallbacks to sequential: {fallback_count}

Next steps:
  /dr-status   — see overall progress (includes mode stats)
  /dr-review   — validate data quality for this phase
  /dr-run      — continue to next phase
```

**Auto-improve modes:** Also show researcher version history:
```
Researcher evolution: v1 ({score1}%) → v2 ({score2}%) → ...
Decision: {kept or reverted}
```

## Error Handling

- If CLAUDE.md is missing, tell the user to run `/dr-new` first.
- If todo.md is missing, tell the user to run `/dr-new` first.
- If a subagent crashes without output, mark the task `[!]` and continue.
- If a data file already exists, the subagent should merge/append, not overwrite.
- If parallel-plan.md is missing when running in parallel mode, spawn dr-planner first (Step 0.5).
- If the planner recommends SEQUENTIAL ONLY, suggest `/dr-run sequential` and do not proceed in parallel.
