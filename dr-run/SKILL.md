---
name: dr-run
description: Execute the next batch of research tasks autonomously. Reads todo.md, spawns subagents for each task, verifies output, and loops until the current phase is complete or a stop condition is hit. Use after /dr:new has scaffolded the project.
---

You are executing an autonomous research loop. Read CLAUDE.md for the full execution protocol.

## Execution Loop

### Step 1: Find Current Phase

Read `.research/ROADMAP.md`. Find the current phase — the first phase with status `NOT_STARTED` or `IN_PROGRESS`. If all phases are `COMPLETE`, tell the user and suggest `/dr:report`.

### Step 2: Activate Phase

Update the current phase status to `IN_PROGRESS` in `.research/ROADMAP.md`.

### Step 3: Find Next Task

Read `todo.md`. Find the first unchecked task matching `- [ ] P{current_phase}.*`. If no unchecked tasks remain for this phase, go to Step 11.

### Step 4: Load Schema

Read `.research/ARCHITECTURE.md` to get the output schema for this phase's entity type.

### Step 5: Spawn Researcher

Spawn the `dr-researcher` subagent with this prompt:

```
Your task: {task description from todo.md}
Output schema: {relevant schema from ARCHITECTURE.md}
Write to: {file path specified in the task}
Follow the data rules in CLAUDE.md. You have a 10-minute time budget.
```

Use the Agent tool with `subagent_type: "general-purpose"` and include the full task context.

### Step 6: Verify Output

Check the subagent's output:
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

### Step 9: Log to Changelog

Append to `.research/CHANGELOG.md`:
```
[ISO timestamp] | {task_id} | DONE or FAIL | {output file} | {one-line summary of what was found or why it failed}
```

### Step 10: Checkpoint

Every 5 tasks, write a checkpoint entry to `.research/CHANGELOG.md`:
```
[ISO timestamp] | CHECKPOINT | completed: {N}, failed: {N}, remaining: {N} | patterns: {observations} | recommendation: continue/stop
```

Check stop conditions:
- All phase tasks checked → go to Step 11
- 3+ hours elapsed → stop with time warning
- 3 consecutive failures → stop with failure warning
- Rate limited 3 times → stop with rate limit warning

If not stopped, go to Step 3.

### Step 11: Phase Complete

When all tasks for the current phase are checked:
1. Update phase status to `COMPLETE` in `.research/ROADMAP.md`
2. Write phase completion entry to CHANGELOG
3. Count done `[x]` and failed `[!]` tasks for this phase

Print:
```
Phase {N} complete. {done} done, {failed} failed.

Next steps:
  /dr:status   — see overall progress
  /dr:review   — validate data quality for this phase
  /dr:improve  — run self-improvement loop on the researcher
  /dr:run      — continue to next phase
```

## Error Handling

- If CLAUDE.md is missing, tell the user to run `/dr:new` first.
- If todo.md is missing, tell the user to run `/dr:new` first.
- If a subagent crashes without output, mark the task `[!]` and continue.
- If a data file already exists, the subagent should merge/append, not overwrite.
