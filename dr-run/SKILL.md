---
name: dr-run
description: Execute the next batch of research tasks autonomously. Reads todo.md, spawns subagents for each task, verifies output, and loops until the current phase is complete or a stop condition is hit. Use after /dr:new has scaffolded the project.
---

You are executing an autonomous research loop. Read CLAUDE.md for the full execution protocol.

Accepts an optional mode argument: `/dr:run`, `/dr:run sequential`, `/dr:run parallel`, `/dr:run parallel-auto-improve`, `/dr:run sequential-auto-improve`.

## Execution Loop

### Step 0: Determine Execution Mode

1. Check if the user passed a mode argument: `sequential` | `parallel` | `sequential-auto-improve` | `parallel-auto-improve`
2. If no argument, read `CLAUDE.md` for the "Execution Mode" section and use the default mode listed there
3. If `CLAUDE.md` has no "Execution Mode" section, fall back to `sequential`

Print: `Mode: {mode}`

### Step 0.5: Dependency Analysis (parallel modes only)

If mode is `parallel` or `parallel-auto-improve`:

1. Check if `.research/parallel-plan.md` exists AND was created after the last `todo.md` change (compare file modification times using `stat`)
2. If the plan does not exist or is stale:
   a. Spawn the `dr-planner` subagent to analyze the current phase. Use the Agent tool with `subagent_type: "dr-planner"`.
   b. After the planner returns, **STOP** and show the plan summary to the user.
   c. Print: "Parallel plan written to `.research/parallel-plan.md`. Review it, then run `/dr:run parallel` again to proceed."
   d. If the plan recommends **SEQUENTIAL ONLY**, print: "The planner recommends sequential execution for this phase. Consider running `/dr:run sequential` instead."
   e. **Do NOT proceed automatically** — force the user to confirm by re-running the command after reviewing the plan.
3. If the plan exists and is fresh, re-read `.research/parallel-plan.md` and proceed using its recommended strategy.

### Step 1: Find Current Phase

Read `.research/ROADMAP.md`. Find the current phase — the first phase with status `NOT_STARTED` or `IN_PROGRESS`. If all phases are `COMPLETE`, tell the user and suggest `/dr:report`.

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

Use the Agent tool with `subagent_type: "dr-researcher"` and include the full task context.

**Parallel modes:** Spawn up to 5 `dr-researcher` subagents **in a single message** using multiple Agent tool calls. Each subagent gets the same prompt format as above but for its specific task. If the parallel plan requires merge strategy for RISKY tasks, instruct each subagent to write to `data/{file}.{task_id}.tmp.json` instead of the canonical file.

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
- 3 consecutive failures (sequential) or 3+ failures in one batch (parallel) → fall back to sequential for the rest of the phase, log: `[ISO timestamp] | MODE_FALLBACK | parallel→sequential | reason: {N} failures in batch`
- Rate limited 3 times → stop with rate limit warning

If not stopped, go to Step 3.

### Step 11: Phase Complete

When all tasks for the current phase are checked:
1. Update phase status to `COMPLETE` in `.research/ROADMAP.md`
2. Write phase completion entry to CHANGELOG
3. Count done `[x]` and failed `[!]` tasks for this phase

### Step 11.5: Auto-Improve (auto-improve modes only)

If mode is `sequential-auto-improve` or `parallel-auto-improve`:
1. Before starting the next phase, automatically run the `/dr:improve` loop against the completed phase's data
2. Log to CHANGELOG: `[ISO timestamp] | AUTO_IMPROVE | phase:{N} | old_pass_rate:{X}% | new_pass_rate:{Y}% | decision:{kept or reverted}`
3. If improved: next phase uses the new researcher. If reverted: next phase uses the previous researcher.
4. Continue to the next phase automatically.

### Step 12: Completion Output

Print results based on mode:

**Sequential modes:**
```
Phase {N} complete. {done} done, {failed} failed.

Next steps:
  /dr:status   — see overall progress
  /dr:review   — validate data quality for this phase
  /dr:improve  — run self-improvement loop on the researcher
  /dr:run      — continue to next phase
```

**Parallel modes:** Also show batch-level stats:
```
Phase {N} complete. {done} done, {failed} failed.
Batches: {batch_count} | Fallbacks to sequential: {fallback_count}

Next steps:
  /dr:status   — see overall progress (includes mode stats)
  /dr:review   — validate data quality for this phase
  /dr:run      — continue to next phase
```

**Auto-improve modes:** Also show researcher version history:
```
Researcher evolution: v1 ({score1}%) → v2 ({score2}%) → ...
Decision: {kept or reverted}
```

## Error Handling

- If CLAUDE.md is missing, tell the user to run `/dr:new` first.
- If todo.md is missing, tell the user to run `/dr:new` first.
- If a subagent crashes without output, mark the task `[!]` and continue.
- If a data file already exists, the subagent should merge/append, not overwrite.
- If parallel-plan.md is missing when running in parallel mode, spawn dr-planner first (Step 0.5).
- If the planner recommends SEQUENTIAL ONLY, suggest `/dr:run sequential` and do not proceed in parallel.
