# Session 1: Landscape Mapping

## Objective

Complete Phase 1 of the research roadmap: identify all players in the target market.

## Before starting

Read these files:
- `.research/ROADMAP.md` — check Phase 1 status
- `todo.md` — find first unchecked P1.xx task
- `.research/CHANGELOG.md` — check for previous progress

## Execution

Run `/dr:run` to begin the autonomous execution loop. The loop will:
1. Find the next unchecked P1 task
2. Spawn the researcher subagent
3. Verify output
4. Mark done or failed
5. Log to changelog
6. Repeat

## Termination conditions

Stop when:
- All P1 tasks are checked (done or failed)
- 3 hours have elapsed
- 3 consecutive failures occur
- Rate limited 3 times

## On completion

Write a session summary to `.research/CHANGELOG.md`:
```
[timestamp] | SESSION_END | Phase 1 | completed: N, failed: N, remaining: N | key observations
```

Then suggest: `/dr:status` to see progress, `/dr:review` to check quality, `/dr:run` to continue.
