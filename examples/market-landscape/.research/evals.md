# Evaluation Criteria

Binary yes/no per entry. Used by /dr-improve to score research quality.

## Default Criteria

1. **has_all_required_fields** — Does the entry have all required schema fields populated (name, category, website, sources, status, last_updated)?
2. **has_source_url** — Does the entry have at least 1 source URL in the sources array?
3. **valid_status** — Is the status field set to a valid value (COMPLETE, INCOMPLETE, or INACCESSIBLE)?
4. **claims_sourced** — Are funding amounts and founding dates backed by a source URL? (NOT_FOUND is acceptable; unsourced specific values are not)
5. **own_website_in_sources** — Is the entity's own website listed in the sources array (if status is not INACCESSIBLE)?

## Domain-Specific Criteria

6. **has_category** — Is the category one of the defined categories in ARCHITECTURE.md?
7. **has_description** — Does the entry have a non-empty description that is not NOT_FOUND?
8. **funding_consistency** — If funding_total and funding_stage are both present, are they consistent? (e.g., $50M total with Series A is plausible; $500M total with Seed is not)

## Adversary-Aware Criteria

These criteria are active only when adversary sidecar files (`data/*.adversary.json`) exist. If no sidecar files are present, skip these criteria.

9. **adversary_survival_rate** — Is the adversary survival rate >= 0.7? Formula: (confirmed + weakened) / (confirmed + weakened + refuted). Exclude pending and unverifiable from denominator.
10. **provenance_completeness** — Do >= 90% of provenance_fields (funding_total, employee_count, founded_year) have full provenance envelopes with no nulls in required fields?
11. **source_support_rate** — Do >= 80% of provenance claims use extraction_method of direct_quote or paraphrase (not inference)?
