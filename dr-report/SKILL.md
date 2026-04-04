---
name: dr-report
description: Synthesize research findings into the final deliverable. Reads all data files, generates executive summary, key findings, gaps, and recommendations. Use when all phases are complete.
---

You are synthesizing research findings into the final deliverable.

## Steps

### 1. Load Context

Read `.research/PROJECT.md` for:
- The original research question / mission
- The decision context (what the user will do with this)
- The desired artifact type (strategic brief, landscape map, scorecard, etc.)

### 2. Check Completeness

Read `.research/ROADMAP.md`. For each phase:
- If COMPLETE, proceed
- If IN_PROGRESS or NOT_STARTED, warn the user: "Phase {N} is not complete. The report will have gaps. Continue anyway? Or run `/dr:run` first."

### 3. Load All Data

Read every file in `data/`. For each:
- Parse the entries
- Note total count, completeness rate, CONFLICTING/UNVERIFIED values
- Group by category or phase as appropriate

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

### 6. Print Summary

After writing the report:

```
Report written to output/final-report.md

{word count} words, {section count} sections
{entity count} entities across {category count} categories
{source count} unique sources referenced

The report directly answers: "{original research question}"
```

## Error Handling

- If no data files exist, tell the user to run `/dr:run` first.
- If PROJECT.md is missing, tell the user to run `/dr:new` first.
- If data is sparse (<10 entries total), warn that findings may not be representative.
