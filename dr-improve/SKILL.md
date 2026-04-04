---
name: dr-improve
description: Self-improvement loop for research quality. Evaluates completed research tasks against binary eval criteria, identifies failure patterns, mutates the researcher subagent instructions, re-runs a sample of failed tasks, and keeps the improved version if score goes up.
---

You are running a self-improvement loop on the research process. The researcher subagent's instructions are the "trainable parameter." Eval pass rate is the metric.

Accepts arguments:
- `/dr:improve 10` — set sample size to 10 (default: 5)
- `/dr:improve --criteria` — interactively add new eval criteria
- `/dr:improve --cycles 3` — run multiple improvement cycles

## STEP 1: Load Eval Criteria

Read `.research/evals.md`. Each criterion is binary: PASS or FAIL per entry. Parse them into a checklist.

If `--criteria` flag is passed, ask the user to add new criteria before proceeding. Append to evals.md.

## STEP 2: Score Current Performance

Read all files in `data/`. For each entry in each file:
- Run every eval criterion → PASS or FAIL
- Record results per entry and per criterion

Calculate:
- Overall pass rate (entries where ALL criteria pass / total entries)
- Per-criterion pass rate
- Total entries evaluated

Save results to `.research/eval-history/baseline.json` (first run) or `.research/eval-history/iteration-{N}.json` (subsequent runs).

Format:
```json
{
  "timestamp": "ISO date",
  "iteration": 0,
  "overall_pass_rate": 0.68,
  "total_entries": 45,
  "per_criterion": {
    "has_all_required_fields": 0.85,
    "has_source_url": 0.92,
    "valid_status": 0.98,
    "claims_sourced": 0.62,
    "own_website_in_sources": 0.55
  },
  "entries_evaluated": 45,
  "entries_passing": 31
}
```

## STEP 3: Analyze Failures

Spawn the `dr-evaluator` subagent with:

```
Analyze these eval results:
- Overall pass rate: {rate}
- Per-criterion rates: {rates}
- Total entries: {count}

Read the data files in data/ and the eval criteria in .research/evals.md.

Answer:
1. Which criteria fail most frequently?
2. Are failures clustered by category, type, or data source?
3. What do passing entries do differently from failing entries?
4. What specific changes to the researcher's instructions would fix the top 3 failure patterns?

Return structured JSON with top_failure_patterns and suggested_improvements.
```

## STEP 4: Mutate Researcher

1. Read current `.claude/agents/dr-researcher.md` (project-local) or fall back to the installed `agents/dr-researcher.md`
2. Save a backup copy to `.research/eval-history/researcher-v{N}.md`
3. Apply surgical changes targeting the top 3 failure patterns identified in Step 3
4. Changes must be concrete instructions, not vague advice:
   - BAD: "Be more thorough when searching"
   - GOOD: "Always search for '{entity name} funding' and '{entity name} crunchbase' as separate queries. Extract the funding amount from the first result that contains a dollar figure."
5. Change 1-3 instructions per cycle. Do not rewrite the entire file.
6. Write the updated researcher to `.claude/agents/dr-researcher.md`

## STEP 5: Re-run Sample

1. Determine sample size (default 5, or from argument)
2. Select tasks to re-run:
   - First, pick tasks marked `[!]` (failed) in todo.md
   - If not enough `[!]` tasks, pick tasks whose entries had the lowest eval scores
3. For each selected task:
   - Reset the task from `[!]` or `[x]` to `[ ]` in todo.md
   - Back up the existing data entry
   - Run using the updated researcher (same process as /dr:run Step 5)
   - Verify output (same process as /dr:run Step 6)
   - Mark `[x]` or `[!]` in todo.md

## STEP 6: Score Again

Run the same eval criteria from Step 2 on the full dataset (including the re-run entries). Save to `.research/eval-history/iteration-{N}.json`.

## STEP 7: Keep or Revert

Compare new overall pass rate to previous:

**New score > old score → KEEP**
```
Improvement: {old}% → {new}% (+{delta}%)
Changes kept:
  - {change 1}
  - {change 2}
Researcher version: v{N}
```

**New score <= old score → REVERT**
```
No improvement: {old}% → {new}%
Reverting researcher to v{N-1}.
Changes discarded:
  - {change 1}
  - {change 2}
```
Restore the backup researcher from `.research/eval-history/researcher-v{N-1}.md`.

Log to `.research/CHANGELOG.md`:
```
[timestamp] | IMPROVE | {old}% → {new}% | kept/reverted | {summary of changes}
```

## STEP 8: Suggest Next Action

Based on the current pass rate:

- **< 70%:** "Pass rate is below 70%. Recommend running `/dr:improve` again to target remaining failure patterns."
- **70-85%:** "Pass rate is in the acceptable range. Continue research with `/dr:run` or improve further with `/dr:improve`."
- **> 85%:** "Pass rate is strong. Ready for `/dr:report` to synthesize findings, or continue with `/dr:run`."

If `--cycles` flag was passed and cycles remain, automatically go back to Step 2.

## Error Handling

- If evals.md is missing, tell the user to run `/dr:new` first.
- If no data files exist, tell the user to run `/dr:run` first to generate data before improving.
- If the researcher file doesn't exist, create one from the default template.
