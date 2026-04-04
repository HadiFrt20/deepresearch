---
name: dr-review
description: Validate research data quality. Checks output files against schemas, flags NOT_FOUND clusters, incomplete entries, conflicting sources, and suspicious data. Use after a phase completes.
---

You are auditing research data quality. Read the schemas and check every data file against them.

## Steps

### 1. Load Schemas

Read `.research/ARCHITECTURE.md` to get the JSON schemas for each entity type. Note all required fields and valid values.

### 2. Audit Each Data File

For each file in `data/`:

**Required Fields Check:**
- List entries missing any required field (name, category, website, sources, status, last_updated)
- List entries where required fields are empty strings or null

**Status Distribution:**
- Count entries by status: COMPLETE, INCOMPLETE, INACCESSIBLE
- Flag entries with no status field

**NOT_FOUND Analysis:**
- Per field, count how many entries have "NOT_FOUND"
- Calculate NOT_FOUND rate per field
- Flag fields with >40% NOT_FOUND — these may need a different search strategy

**Source Quality:**
- Entries with zero source URLs (critical)
- Entries missing the entity's own website in sources
- Duplicate source URLs across entries (may indicate copy-paste errors)

**Data Integrity:**
- Entries with CONFLICTING values — list each conflict
- Entries with UNVERIFIED values — count per field
- Duplicate entries (same name or same website)
- Entries where status is COMPLETE but fields are NOT_FOUND (inconsistent)

### 3. Quality Report

Print findings grouped by severity:

```
## Data Quality Report

### Critical (must fix before report)
- 3 entries missing source URLs
- 1 entry with no status field
- 2 entries marked COMPLETE but have NOT_FOUND fields

### Warning (should investigate)
- funding_amount: 45% NOT_FOUND rate
- 4 entries with UNVERIFIED claims
- 2 potential duplicates: "Company A" appears twice

### Info
- 18 COMPLETE, 5 INCOMPLETE, 2 INACCESSIBLE
- 3 CONFLICTING values found (listed below)
- Average sources per entry: 2.3
```

### 4. Suggest Actions

Based on findings:
- Critical issues → "Fix these before running `/dr:report`. Consider `/dr:run` to retry failed tasks."
- High NOT_FOUND rates → "Run `/dr:improve` to optimize the researcher's search strategy for these fields."
- Many INCOMPLETE → "Run `/dr:run` to continue. Some entities may need manual research."
- Clean data → "Data looks good. Run `/dr:report` to synthesize findings."

## Error Handling

- If no data files exist, tell the user to run `/dr:run` first.
- If ARCHITECTURE.md is missing, tell the user to run `/dr:new` first.
