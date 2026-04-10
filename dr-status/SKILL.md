---
name: dr-status
description: Show research progress dashboard. Displays completed/failed/remaining task counts, phase status, data completeness, and recent changelog entries.
---

You are displaying a research progress dashboard. Read the project files and present a clear status overview.

## Steps

### 1. Task Counts

Read `todo.md` and count:
- `[x]` — completed tasks
- `[!]` — failed tasks
- `[ ]` — remaining tasks
- Total tasks

Break down by phase (P1.xx, P2.xx, etc.).

### 2. Phase Status

Read `.research/ROADMAP.md` and show each phase with its status (NOT_STARTED, IN_PROGRESS, COMPLETE).

### 3. Data Completeness

For each file in `data/`:
- Count total entries
- For each required schema field, calculate the percentage that are NOT `"NOT_FOUND"`, `null`, or empty
- Flag fields with >30% NOT_FOUND rate
- Note any CONFLICTING or UNVERIFIED values

### 4. Recent Activity

Show the last 10 entries from `.research/CHANGELOG.md`.

### 5. Trust Score

Read all sidecar files (`data/*.adversary.json`). For each sidecar file:
- Parse the JSON array of adversary verdicts
- **Error handling:** If a sidecar file contains malformed JSON, skip it, note `"[WARNING] Malformed sidecar: {filename}"` in the dashboard output, and continue processing other files
- Count verdicts by type: confirmed, refuted, weakened, unverifiable, and count claims still `pending` (claims with provenance but no matching sidecar verdict)

Calculate:
- `adversary_survival_rate`: `(confirmed + weakened) / (confirmed + weakened + refuted)`. Exclude pending and unverifiable from denominator.
- `mean_survival_score`: average `survival_score` across all adversary-reviewed claims (those with verdict != pending)
- `pending_count`: claims with provenance envelopes but no adversary verdict yet
- `total_reviewed`: confirmed + weakened + refuted + unverifiable

Find the top 5 most-attacked entities by count of `refuted` + `weakened` verdicts. For each, find the claim with the lowest `survival_score`.

**Display logic:**
- If no sidecar files exist AND provenance exists in data files → show `Trust: -- (adversary not yet run)`
- If no sidecar files exist AND no provenance exists → show nothing (trust section omitted)
- If `--no-adversary` was used (detected by: provenance exists in data files but zero sidecar files AND the CHANGELOG contains an entry with `Adversary: disabled`) → show `Adversary: disabled`
- If sidecar files exist → show full trust dashboard

### 6. Dashboard Output

Print a formatted dashboard:

```
## Research Progress

Phase | Status      | Done | Failed | Remaining
------|-------------|------|--------|----------
P1    | COMPLETE    |   18 |      2 |         0
P2    | IN_PROGRESS |    5 |      1 |        14
P3    | NOT_STARTED |    0 |      0 |        20
P4    | NOT_STARTED |    0 |      0 |        25

Total: 23 done, 3 failed, 59 remaining (85 total)
Overall: 27% complete

## Data Completeness

File                    | Entries | Completeness
------------------------|---------|-------------
data/companies-p1.json  |      18 | 82%
data/companies-p2.json  |       5 | 75%

Fields below 70%: funding_amount (45%), employee_count (62%)

## Trust Score

Trust: 87% survived | Score: 0.74
Adversary: 45/52 claims reviewed (7 pending)

## Most Attacked

Entity           | Refuted | Weakened | Lowest claim
-----------------|---------|----------|-----------------------------
Regie.ai         |       2 |        1 | "funding: $30M" (0.12)
Orum             |       0 |        2 | "employee_count: 60" (0.42)

## Recent Activity

[last 10 changelog entries]
```

### 7. Suggest Next Action

Based on the state:
- Unchecked tasks remain → "Run `/dr-run` to continue execution."
- Phase just completed → "Run `/dr-review` to validate data quality, then `/dr-improve` to optimize the researcher."
- All phases complete → "Run `/dr-report` to synthesize findings."
- High failure rate (>30%) → "Run `/dr-improve` to analyze failures and improve the researcher."

## Error Handling

- If todo.md is missing, tell the user to run `/dr-new` first.
- If no data files exist yet, show task counts and phases only.
