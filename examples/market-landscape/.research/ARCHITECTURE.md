# Architecture Decisions

## ADR-1: Breadth vs Depth

**Decision:** Breadth-first within each category, then depth pass on promising companies.

**Rationale:** We need category coverage to identify whitespace. Deep research on all 50+ companies would take too long. Phase 1 captures the landscape, Phase 3 goes deep on top candidates.

## ADR-2: Verification Strategy

**Decision:** Cross-reference key claims (funding, employee count, founding date) across 2+ sources. Single-source facts marked UNVERIFIED.

**Rationale:** Funding data is frequently outdated or inaccurate. Crunchbase, PitchBook, and company websites often disagree. We need to flag conflicts rather than pick one.

## ADR-3: Dead End Handling

**Decision:** Flag as INACCESSIBLE after 3 search attempts. Move on. Retry in a later pass if time permits.

**Rationale:** Some stealth-mode startups have minimal web presence. Better to flag and move on than spend 30 minutes on a dead end.

## ADR-4: Output Format

**Decision:** JSON arrays with the schema below. One file per phase, entries appended.

### Company Profile Schema

```json
{
  "name": "string (required)",
  "category": "string (required) — one of: AI SDR, Email Automation, Call Automation, Meeting Scheduling, Lead Scoring, Pipeline Intelligence, Multi-channel",
  "website": "string (required) — primary company URL",
  "description": "string — one-line company description",
  "founded_year": "number | 'NOT_FOUND' | 'UNVERIFIED: {year}'",
  "headquarters": "string | 'NOT_FOUND'",
  "employee_count": "string | 'NOT_FOUND' | 'UNVERIFIED: {count}' | 'CONFLICTING: {v1} vs {v2}'",
  "funding_total": "string | 'NOT_FOUND' | 'UNVERIFIED: {amount}' | 'CONFLICTING: {v1} vs {v2}'",
  "funding_stage": "string | 'NOT_FOUND' — latest round (Seed, Series A, etc.)",
  "key_investors": "array of strings | ['NOT_FOUND']",
  "pricing_model": "string | 'NOT_FOUND' — freemium, per-seat, usage-based, enterprise-only",
  "key_features": "array of strings — top 3-5 features",
  "target_market": "string | 'NOT_FOUND' — SMB, mid-market, enterprise",
  "notable_customers": "array of strings | ['NOT_FOUND']",
  "differentiator": "string | 'NOT_FOUND' — what makes them unique",
  "sources": "array of URLs (required, min 1)",
  "status": "'COMPLETE' | 'INCOMPLETE' | 'INACCESSIBLE' (required)",
  "last_updated": "string — ISO date (required)"
}
```

### Field Requirements

- `name`, `category`, `website`, `sources`, `status`, `last_updated` are always required
- All other fields accept `"NOT_FOUND"` as a valid value
- `"UNVERIFIED: {value}"` for single-source claims
- `"CONFLICTING: {v1} vs {v2}"` when sources disagree

## Provenance

provenance_fields: ["funding_total", "employee_count", "founded_year"]

These 3 fields get full provenance envelopes (12 fields each) with adversarial verification.
All other factual fields get a simple `"sourced": true/false` flag.
