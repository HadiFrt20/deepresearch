---
name: dr-report
description: Synthesize research findings into the final deliverable. Reads all data files, generates executive summary, key findings, gaps, and recommendations. Use when all phases are complete.
---

You are synthesizing research findings into the final deliverable.

Accepts optional arguments: `/dr-report`, `/dr-report --provenance`.

## Steps

### 1. Load Context

Read `.research/PROJECT.md` for:
- The original research question / mission
- The decision context (what the user will do with this)
- The desired artifact type (strategic brief, landscape map, scorecard, etc.)

### 2. Check Completeness

Read `.research/ROADMAP.md`. For each phase:
- If COMPLETE, proceed
- If IN_PROGRESS or NOT_STARTED, warn the user: "Phase {N} is not complete. The report will have gaps. Continue anyway? Or run `/dr-run` first."

### 3. Load All Data

Read every file in `data/`. For each:
- Parse the entries
- Note total count, completeness rate, CONFLICTING/UNVERIFIED values
- Group by category or phase as appropriate

**Merge adversary sidecar files:** Also read any `data/*.adversary.json` sidecar files. Merge adversary verdicts into the data entries by matching `task_id` and `field_name` to the provenance envelopes.
- **Error handling:** If a sidecar file contains malformed JSON, skip it, log `"[WARNING] Skipped malformed sidecar: {filename}"`, and continue processing other files. Do not crash.
- Update `adversary_verdict`, `adversary_evidence`, `confidence_delta`, `verified_at`, and `survival_score` in each matching provenance envelope.

### 4. Generate Report

Write `output/final-report.md` with this structure:

```markdown
# {Research Title}

Generated: {ISO date}
Source: deepresearch autonomous research project

## Executive Summary

{2-3 paragraphs directly answering the research question from PROJECT.md.
This is the most important section. Be specific and actionable.
Reference data points and counts.}

## Methodology

- Research approach: {from ARCHITECTURE.md ADRs}
- Total entities researched: {count}
- Data sources: {count unique source URLs}
- Phases completed: {N of M}
- Overall data completeness: {percentage}

## Key Findings

### {Phase 1 Title}

{Findings from Phase 1 data. Include specific numbers, top entries, patterns.
Reference the data files for details.}

### {Phase 2 Title}

{Findings from Phase 2 data.}

{...repeat for each phase...}

**Adversary highlights in findings:** When presenting claims in Key Findings sections:
- Claims with `adversary_verdict: "refuted"` → mark with `[REFUTED]` and strikethrough: `~~claim text~~ [REFUTED]`
- Claims with `adversary_verdict: "weakened"` → mark with `[WEAKENED]` and show adversary counter-evidence inline: `claim text [WEAKENED — adversary found: "counter-evidence quote"]`
- Claims with `adversary_verdict: "confirmed"` → present normally (no special marking)
- Claims with `adversary_verdict: "pending"` or `"unverifiable"` → present normally

At the end of each Key Findings section, add a **Contested Claims** subsection listing all refuted/weakened claims with their adversary evidence:

```markdown
#### Contested Claims

| Claim | Verdict | Adversary Evidence | Survival Score |
|-------|---------|-------------------|----------------|
| Artisan AI raised $25M | REFUTED | "Crunchbase shows $12M total funding" | 0.00 |
| Regie.ai has 80 employees | WEAKENED | "LinkedIn shows ~120 employees" | 0.42 |
```

If no adversary sidecar files exist, skip all adversary highlights.

## Patterns and Insights

{Cross-cutting observations that span multiple phases.
What trends emerged? What was surprising?
What does the data suggest that wasn't in the original question?}

## Gaps and Limitations

- {N} entries marked INCOMPLETE — could not find full data
- {N} entries marked INACCESSIBLE — sources were unavailable
- Fields with high NOT_FOUND rates: {list}
- {N} CONFLICTING values unresolved
- {Any phases not completed}

## Recommendations

{Based on the decision context from PROJECT.md.
What should the user do with this information?
Prioritized, specific, actionable.}

## Appendix

### Data Sources

{Count of unique URLs. Top domains referenced.}

### Completeness by Field

| Field | Complete | NOT_FOUND | UNVERIFIED | CONFLICTING |
|-------|----------|-----------|------------|-------------|
| ...   | ...      | ...       | ...        | ...         |

### Research Quality

- Eval pass rate: {from latest eval-history file, if exists}
- Researcher iterations: {count of versions in eval-history}
```

### 5. Adapt to Artifact Type

Based on the done condition from PROJECT.md:
- **Strategic brief:** Focus on executive summary and recommendations
- **Landscape map:** Include a category breakdown table with key metrics per entity
- **Opportunity scorecard:** Add a scoring matrix with weighted criteria
- **Spreadsheet:** Additionally generate `output/data-export.json` with all entries merged and normalized

### 6. Provenance Export (--provenance flag)

If the user passed `--provenance`:

Write `output/provenance-chain.json` alongside the markdown report. This is the full epistemic audit trail.

Structure:
```json
{
  "generated_at": "2026-04-10T14:30:00Z",
  "trust_score": 0.74,
  "survival_rate": 0.87,
  "claims": {
    "P1.01:funding_total": {
      "claim_text": "Artisan AI raised $12M",
      "source_url": "https://crunchbase.com/organization/artisan-ai",
      "extraction_method": "direct_quote",
      "cross_ref_count": 2,
      "cross_ref_urls": ["https://techcrunch.com/artisan-ai-funding"],
      "confidence_score": 0.9,
      "volatility_class": "medium",
      "adversary_verdict": "confirmed",
      "adversary_evidence": null,
      "verified_at": "2026-04-10T14:32:00Z",
      "survival_score": 0.9
    }
  }
}
```

- Key format: `{task_id}:{field_name}` for easy lookup
- Flatten all provenance envelopes from all data files + sidecar verdicts into one exportable artifact
- `trust_score`: mean survival_score across all adversary-reviewed claims
- `survival_rate`: (confirmed + weakened) / (confirmed + weakened + refuted)
- If no adversary data exists, set `trust_score` and `survival_rate` to `null`

### 7. Print Summary

After writing the report:

```
Report written to output/final-report.md

{word count} words, {section count} sections
{entity count} entities across {category count} categories
{source count} unique sources referenced
Trust: {survival_rate}% survived | Score: {trust_score} (or "N/A" if no adversary data)

The report directly answers: "{original research question}"
```

If `--provenance` was passed, also print: `Provenance chain written to output/provenance-chain.json ({claim count} claims)`

## Error Handling

- If no data files exist, tell the user to run `/dr-run` first.
- If PROJECT.md is missing, tell the user to run `/dr-new` first.
- If data is sparse (<10 entries total), warn that findings may not be representative.
