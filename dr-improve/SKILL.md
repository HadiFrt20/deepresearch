---
name: dr-improve
description: Self-improvement loop for research quality. Evaluates completed research tasks against binary eval criteria, identifies failure patterns, mutates the researcher subagent instructions, re-runs a sample of failed tasks, and keeps the improved version if score goes up.
---

You are running a self-improvement loop on the research process. The researcher subagent's instructions are the "trainable parameter." Eval pass rate is the metric.

Accepts arguments:
- `/dr-improve` — single improvement cycle, default sample size 5
- `/dr-improve 10` — set sample size to 10
- `/dr-improve --criteria` — add new eval criteria before improving
- `/dr-improve --cycles 3` — run 3 improvement cycles
- `/dr-improve 10 --criteria --cycles 3` — combine: sample 10, add criteria, 3 cycles

### Argument Parsing

1. Check for a bare number argument → set sample_size (default: 5)
2. Check for `--criteria` flag → set add_criteria = true
3. Check for `--cycles N` → set cycle_count = N (default: 1)

## STEP 1: Load Eval Criteria

Read `.research/evals.md`. Each criterion is binary: PASS or FAIL per entry. Parse them into a checklist.

If `--criteria` flag is passed, ask the user to add new criteria before proceeding. Append to evals.md.

## STEP 2: Score Current Performance

Read all files in `data/`. For each entry in each file:
- Run every eval criterion → PASS or FAIL
- Record results per entry and per criterion

**Adversary-aware scoring:** Also read any sidecar files (`data/*.adversary.json`). Merge sidecar verdicts with data entries by matching `task_id` and `field_name`.
- If sidecar files exist → include `adversary_survival_rate` as a key metric alongside overall pass rate
- If NO sidecar files exist (zero `*.adversary.json` files) → skip all adversary criteria automatically. The data IS the state — no config flag needed.
- If a sidecar file contains malformed JSON → skip that file, log a warning, continue processing others

Calculate:
- Overall pass rate (entries where ALL criteria pass / total entries)
- Per-criterion pass rate
- Total entries evaluated
- Adversary survival rate (if sidecar files exist): `(confirmed + weakened) / (confirmed + weakened + refuted)`

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
- Adversary survival rate: {rate or "N/A — no sidecar files"}

Read the data files in data/ and the eval criteria in .research/evals.md.
Also read any sidecar files (data/*.adversary.json) and merge verdicts with data entries.

Answer:
1. Which criteria fail most frequently?
2. Are failures clustered by category, type, or data source?
3. What do passing entries do differently from failing entries?
4. What specific changes to the researcher's instructions would fix the top 3 failure patterns?
5. Which claims were refuted or weakened by the adversary? What patterns emerge?

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

6. **Inject Known Vulnerabilities (if sidecar files exist):**
   After identifying failure patterns, also extract "attack patterns" from sidecar files (`data/*.adversary.json`):
   - An attack pattern = a `field_name` + `extraction_method` combination that was `refuted` or `weakened` 2+ times across batches
   - Scan all sidecar verdicts, group by `field_name`, count refuted+weakened per field
   - Take the top 3 attack patterns (by refuted+weakened count)
   - Inject them as "Known vulnerabilities" bullet points in the researcher's `## Data Rules` section, right after the existing rules
   - Format each as: `- **Known vulnerability:** {field} claims using {method} are frequently contested. {specific mitigation advice based on the adversary's counter-evidence}.`
   - **Cap at 5 total Known Vulnerabilities.** If the researcher already has 5, replace the oldest (topmost) vulnerability with the newest one. Oldest = the one that has been there the longest (first in the list).
   - If no sidecar files exist, skip this sub-step entirely.

7. Write the updated researcher to `.claude/agents/dr-researcher.md`

## STEP 5: Re-run Sample

1. Determine sample size (default 5, or from argument)
2. Select tasks to re-run:
   - First, pick tasks marked `[!]` (failed) in todo.md
   - If not enough `[!]` tasks, pick tasks whose entries had the lowest eval scores
3. For each selected task:
   - Reset the task from `[!]` or `[x]` to `[ ]` in todo.md
   - Back up the existing data entry
   - Run using the updated researcher (same process as /dr-run Step 5)
   - Verify output (same process as /dr-run Step 6)
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

- **< 70%:** "Pass rate is below 70%. Recommend running `/dr-improve` again to target remaining failure patterns."
- **70-85%:** "Pass rate is in the acceptable range. Continue research with `/dr-run` or improve further with `/dr-improve`."
- **> 85%:** "Pass rate is strong. Ready for `/dr-report` to synthesize findings, or continue with `/dr-run`."

If `--cycles` flag was passed and cycles remain, automatically go back to Step 2.

## Error Handling

- If evals.md is missing, tell the user to run `/dr-new` first.
- If no data files exist, tell the user to run `/dr-run` first to generate data before improving.
- If the researcher file doesn't exist, create one from the default template.
