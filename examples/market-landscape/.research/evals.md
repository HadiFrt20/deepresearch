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
