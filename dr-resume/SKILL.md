---
name: dr-resume
description: Resume research after a crash or new session. Reads changelog to find last completed task, shows what was done and what remains, then offers to continue.
---

You are helping the user resume a research project after a session break, crash, or context window reset.

## Steps

### 1. Read Changelog

Read `.research/CHANGELOG.md`. Find the last entries to determine:
- Timestamp of last activity
- Last completed task ID
- Last status (DONE, FAIL, CHECKPOINT)
- Any session-end summary

### 2. Find Current Position

Read `todo.md`. Find:
- The last task marked `[x]` or `[!]`
- The first unchecked task `[ ]`
- Count remaining unchecked tasks

### 3. Check Phase Status

Read `.research/ROADMAP.md`. Identify:
- Which phase was in progress
- How many phases are complete
- How many phases remain

### 4. Print Resume Summary

```
## Research Resume

Last session: {timestamp of last changelog entry}
Last completed: {task_id} — {task description}
Phase: {phase number} ({phase status})

Progress:
  Completed: {count}
  Failed: {count}
  Remaining: {count}

Next task: {first unchecked task_id} — {task description}
```

### 5. Check for Issues

- If last 3+ entries are FAIL, warn: "Last session had consecutive failures. Consider `/dr-improve` before continuing."
- If there's a CHECKPOINT with "stop" recommendation, show it.
- If todo.md has tasks marked `[!]`, note: "{N} failed tasks could be retried."

### 6. Suggest Next Action

- Tasks remain in current phase → "Run `/dr-run` to continue from {next task_id}."
- Phase just completed → "Run `/dr-review` to validate, then `/dr-run` for next phase."
- Many failures → "Run `/dr-improve` to analyze failures and optimize the researcher."
- All done → "Run `/dr-report` to synthesize findings."

## Error Handling

- If CHANGELOG.md is empty or missing, tell the user: "No previous session found. Run `/dr-new` to start a new project or `/dr-run` to begin execution."
- If todo.md is missing, tell the user to run `/dr-new` first.
