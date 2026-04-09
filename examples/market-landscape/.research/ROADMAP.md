# Research Roadmap

## Phase 1: Landscape Mapping

- **Question:** Who are the players in the AI sales agent market?
- **Input:** Category definitions from ARCHITECTURE.md
- **Output:** `data/companies-p1.json`
- **Verification:** File exists, 40+ entries, all required fields populated
- **Estimated tasks:** 20
- **Status:** IN_PROGRESS

## Phase 2: Funding and Traction Deep Dive

- **Question:** Who is well-funded, growing fast, and gaining market share?
- **Input:** `data/companies-p1.json`
- **Output:** `data/companies-p2-funding.json`
- **Verification:** File exists, funding data for 80%+ of P1 companies, investor names cross-referenced
- **Estimated tasks:** 15
- **Status:** NOT_STARTED

## Phase 3: Product and Positioning Analysis

- **Question:** How do these products actually differ? What are the real differentiators?
- **Input:** `data/companies-p1.json`, `data/companies-p2-funding.json`
- **Output:** `data/product-analysis-p3.json`
- **Verification:** File exists, feature comparison for top 20 companies, pricing data for 60%+
- **Estimated tasks:** 10
- **Status:** NOT_STARTED

## Phase 4: Gap Analysis and Synthesis

- **Question:** Where are the underserved segments and whitespace opportunities?
- **Input:** All Phase 1-3 data files
- **Output:** `data/gap-analysis-p4.json`, `data/opportunities-p4.json`
- **Verification:** Files exist, 3+ identified gaps with supporting data, opportunity scores assigned
- **Estimated tasks:** 10
- **Status:** NOT_STARTED
