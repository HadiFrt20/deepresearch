---
name: dr-new
description: Set up a new autonomous research project. Interviews you with 11 questions, then generates the full project scaffold (specs, roadmap, tasks, schemas, execution protocol). Use when starting any research project.
---

You are setting up an autonomous multi-session research project. Interview the user, then generate every file needed to run the research autonomously for hours.

## PHASE A: Interview

Ask one question at a time. Wait for the answer before asking the next. There are 11 questions total (A1–A5, A6a–A6c, A7–A9).

IMPORTANT: For questions A3, A6a, A6b, A6c, A7, A8, A9, use the AskUserQuestion tool
to present clickable options. Do NOT ask these as free text. The user should be able
to select with arrow keys and Enter.

For questions A1, A2, A4, A5, use free text prompts because the answers are
open-ended and can't be pre-enumerated.

When using AskUserQuestion, construct it like this:

```
AskUserQuestion({
  question: "What research tools do you have available?",
  options: [
    { label: "Native only", description: "WebSearch + WebFetch, no MCP setup needed" },
    { label: "Firecrawl MCP", description: "Search, scrape, crawl, map — best for deep extraction" },
    { label: "Linkup MCP", description: "Search with structured extraction" },
    { label: "Tavily MCP", description: "Search optimized for AI agents" },
    { label: "Multiple", description: "I'll list them in my next message" }
  ]
})
```

Include a short description for each option to help the user choose.

Wait for the user's selection before proceeding to the next question.

**Fallback:** Before using AskUserQuestion, check if the tool exists in the current session (use ToolSearch to look it up). If not available (older Claude Code version), fall back to numbered free-text prompts ("Pick one: 1, 2, 3, 4"). The fallback should be seamless — users on older versions still get the interview, just without the clickable UI.

**A1 — Mission:** (free text) What are you researching? What specific question are you trying to answer?

**A2 — Decision:** (free text) What will you do with this research? (build a product, write a post, pitch investors, make a career move, other)

**A3 — Done condition:** (AskUserQuestion)
  Question: "What artifact do you want at the end?"
  Options:
    - Strategic brief
    - Spreadsheet
    - Landscape map
    - Opportunity scorecard
    - Multiple (list in next message)

**A4 — Scope:** (free text) How many categories/segments? Geographic scope? Company stage filter? Depth per entity (surface vs deep)?

**A5 — Constraints:** (free text) How many overnight sessions (1-5)? Quality vs breadth? Budget ceiling for API credits?

**A6 — Methodology:** (AskUserQuestion — split into 3 sub-questions)

  **A6a — Depth strategy:**
    Question: "Breadth-first or depth-first?"
    Options:
      - Breadth-first
      - Depth-first
      - Breadth-first then depth
      - Depth-first then breadth

  **A6b — Verification:**
    Question: "Single-pass or cross-referenced?"
    Options:
      - Single-pass (faster)
      - Cross-referenced (more accurate)

  **A6c — Dead ends:**
    Question: "How to handle dead ends?"
    Options:
      - Skip and note
      - Retry once
      - Retry twice then skip
      - Flag for manual review

**A7 — Tools:** (AskUserQuestion)
  Question: "What research tools do you have available?"
  Options:
    - Native only (WebSearch + WebFetch, no MCP setup needed)
    - Firecrawl MCP (search, scrape, crawl, map — best for deep extraction)
    - Linkup MCP (search with structured extraction)
    - Tavily MCP (search optimized for AI agents)
    - Multiple (I'll list them in my next message)

Tell the user: "If you're not sure, pick Native only. You can always add tools later by editing the Tool priority section in CLAUDE.md."

**A8 — Execution mode:** (AskUserQuestion)
  Question: "How should /dr-run execute tasks by default?"
  Options:
    - Sequential (recommended for first run)
    - Sequential + auto-improve
    - Parallel batches
    - Parallel + auto-improve

Tell the user: "If unsure, pick Sequential. You can override per-run with: /dr-run sequential, /dr-run parallel, /dr-run parallel-auto-improve, etc."

**A9 — Permission mode:** (AskUserQuestion)
  Question: "Should /dr-run ask for approval on every tool call?"
  Options:
    - Ask every time (recommended for first-time users)
    - Auto mode — never ask, run fully autonomously

Tell the user: "Pick Auto mode for overnight runs or when you want to start the research and walk away. Pick Ask every time if you want to review each step. You can override at runtime: /dr-run auto or /dr-run ask."

**A10 — Provenance fields:** (AskUserQuestion)
  Question: "Which 3 fields are most important to verify? (These will get full provenance tracking with adversarial verification.)"
  Options: Present the entity-specific fields from the schema you designed in your head during A1-A4 (e.g., funding_total, employee_count, founded_year, pricing_model, revenue, etc.). Show 5-8 options based on the research domain. The user picks exactly 3.

Tell the user: "These 3 fields will get full provenance envelopes — source URL, extraction method, cross-references, confidence score, and adversarial verification. All other factual fields get a simple sourced: true/false flag. You can change this later in .research/ARCHITECTURE.md."

Write the chosen fields to `provenance_fields` in `.research/ARCHITECTURE.md` (see File 2 below).

## PHASE B: Generate Project Scaffold

After all answers, generate these files without asking more questions:

### File 1: `.research/PROJECT.md`

One-page research brief containing:
- Mission statement (from A1)
- Decision context (from A2)
- Success criteria and done condition (from A3)
- Scope definition (from A4)
- Constraints (from A5)
- Methodology (from A6)
- Tool configuration (from A7)

### File 2: `.research/ARCHITECTURE.md`

Methodology ADRs and JSON schemas for each entity type.

Include these ADRs:
- **ADR-1: Breadth vs Depth** — based on A6 answer
- **ADR-2: Verification Strategy** — how facts are cross-referenced
- **ADR-3: Dead End Handling** — based on A6 answer
- **ADR-4: Output Format** — JSON schemas for each entity type

Schema requirements:
- `name` (string, required)
- `category` (string, required)
- `website` (string, required)
- Topic-specific fields based on the research domain
- `sources` (array of URLs, required)
- `status` (enum: "COMPLETE" | "INCOMPLETE" | "INACCESSIBLE", required)
- `last_updated` (ISO date string, required)

Fields that may not be found must accept these sentinel values:
- `"NOT_FOUND"` — could not locate this information
- `"UNVERIFIED: {value}"` — single-source fact, not cross-referenced
- `"CONFLICTING: {v1} vs {v2}"` — sources disagree

At the end of ARCHITECTURE.md, add a provenance section:

```markdown
## Provenance

provenance_fields: ["{A10_field_1}", "{A10_field_2}", "{A10_field_3}"]

These 3 fields get full provenance envelopes (12 fields each) with adversarial verification.
All other factual fields get a simple `"sourced": true/false` flag.
```

### File 3: `.research/ROADMAP.md`

4-6 phases. Each phase contains:
- Question it answers
- Input dependencies
- Output file path
- Verification criteria (file exists + min entries + required fields populated)
- Estimated number of tasks
- Status: `NOT_STARTED`

### File 4: `.research/CHANGELOG.md`

```
# Research Changelog

Format: [timestamp] | [task_id] | [status] | [output_file] | [notes]

---
```

### File 5: `.research/evals.md`

Binary evaluation criteria for research quality (used by /dr-improve).

Default criteria:
1. Does the entry have all required schema fields populated?
2. Does the entry have at least 1 source URL?
3. Is the status field set to a valid value (COMPLETE, INCOMPLETE, or INACCESSIBLE)?
4. Are funding/date claims backed by a source?
5. Is the entity's own website listed in sources?

Add domain-specific criteria based on the research topic.

Add these adversary-aware criteria (active only when adversary sidecar files exist):
6. **adversary_survival_rate** — Is the adversary survival rate >= 0.7? (confirmed + weakened) / (confirmed + weakened + refuted). Skip if no sidecar files.
7. **provenance_completeness** — Do >= 90% of provenance_fields have full provenance envelopes with no nulls?
8. **source_support_rate** — Do >= 80% of provenance claims use direct_quote or paraphrase (not inference)?

### File 6: `.claude/agents/dr-planner.md`

Project-local dependency analysis subagent. Copy the default planner from the installed agents.

IMPORTANT: The subagent frontmatter MUST use `disallowedTools` (denylist), NOT `tools` (allowlist). An allowlist blocks MCP tools from being inherited. Use this frontmatter:

```yaml
---
name: dr-planner
description: Analyzes research task dependencies and recommends whether parallelization is safe. Spawned by /dr-run before running in parallel mode for the first time.
disallowedTools:
  - Edit
model: sonnet
---
```

### File 7: `CLAUDE.md`

Execution protocol containing:

**Mission** — 1 paragraph from PROJECT.md

**Execution Mode** — Add this section near the top, right after Mission:

```
## Execution Mode

Default mode: {A8_ANSWER — one of: sequential, sequential-auto-improve, parallel, parallel-auto-improve}

Supported modes:
- `sequential` — one task at a time, safest, easiest to debug
- `sequential-auto-improve` — sequential + /dr-improve runs automatically at each phase boundary
- `parallel` — batches of 5 tasks in parallel within a phase; phase boundaries remain sequential; requires dependency analysis
- `parallel-auto-improve` — parallel + auto-improve at phase boundaries

Runtime override: user can say "run in parallel mode" or pass /dr-run parallel at any time.

Parallel mode safety: before first parallel run in a project, /dr-run spawns a dr-planner subagent to analyze task dependencies and flag any conflicts. If the plan is unsafe, user is asked to write dependency annotations in todo.md before proceeding.
```

Map A8 answers to mode names: Sequential → sequential, Sequential + auto-improve → sequential-auto-improve, Parallel batches → parallel, Parallel + auto-improve → parallel-auto-improve.

**Permission Mode** — Add this section right after Execution Mode:

```
## Permission Mode

Default: {A9_ANSWER — one of: auto, ask}

When permission mode is "auto":
- All tool calls proceed without user approval
- Research runs end-to-end without interruption
- User can override per-run: /dr-run ask

When permission mode is "ask":
- User approves each web search, fetch, and file write
- Slower but lets you inspect each step
- User can override per-run: /dr-run auto
```

Map A9 answers: "Ask every time" → ask, "Auto mode" → auto.

**Bootstrap** — "Read these files first: .research/ROADMAP.md, todo.md, .research/CHANGELOG.md"

**Execution protocol:**
1. Find first unchecked task in todo.md
2. Spawn dr-researcher subagent with task description, output schema, output file path
3. Verify output: file exists, valid JSON/markdown, required fields populated, sources array non-empty, status field set
4. Pass → mark `[x]` in todo.md; Fail → mark `[!]` in todo.md
5. Append to CHANGELOG: `[ISO timestamp] | [task_id] | DONE/FAIL | [file] | [summary]`
6. Every 5 tasks: write checkpoint to CHANGELOG (completed/failed/remaining counts, patterns, continue/stop recommendation)
7. Repeat from step 1
8. **At phase boundary**: automatically run /dr-improve to evaluate and refine research quality before starting next phase (auto-improve modes only)

**Stop conditions:**
- Phase complete (all tasks checked)
- 3+ hours elapsed
- Rate limited 3 times
- No unchecked tasks remain

**Anti-hallucination rules:**
- Never invent data — NOT_FOUND over guessing
- Source URLs required for every entry
- INACCESSIBLE when site is down/blocked, NOT_FOUND when info doesn't exist
- CONFLICTING when sources disagree — include both values
- UNVERIFIED for single-source claims

**Tool priority** — ADAPT BASED ON A7 ANSWER:
- Native (a): `WebSearch → WebFetch`. Add note: "For deeper research, consider adding Firecrawl MCP."
- Firecrawl (b): `Firecrawl search → Firecrawl scrape → Firecrawl crawl → Firecrawl map → WebFetch → WebSearch`
- Linkup (c): `Linkup search → WebFetch → WebSearch`
- Tavily (d): `Tavily search → WebFetch → WebSearch`
- Multiple (e): combine selected tools in priority chain, `WebFetch + WebSearch` always last

**Session handoff:** Write session summary to CHANGELOG at session end.

**Self-improvement:** After each phase completes, run /dr-improve to evaluate and refine the researcher subagent.

### File 8: `todo.md`

Decompose each roadmap phase into 15-25 atomic tasks. Target 80-120 tasks total.

Format: `- [ ] P{phase}.{nn} | {description with search strategy and output file path}`

Each task should:
- Take 5-10 minutes
- Have a concrete file output
- Be independently verifiable
- Include the search strategy (what to search for, where to look)

### File 9: `.claude/agents/dr-researcher.md`

Project-local researcher subagent.

IMPORTANT: The subagent frontmatter MUST use `disallowedTools` (denylist), NOT `tools` (allowlist). An allowlist blocks MCP tools like Firecrawl, Linkup, and Tavily from being inherited. Use this frontmatter:

```yaml
---
name: dr-researcher
description: Executes a single research micro-task with structured output. Spawned by /dr-run.
disallowedTools:
  - Edit
model: sonnet
---
```

ADAPT INSTRUCTIONS BASED ON A7:
- Native (a): "Search using WebSearch with at least 3 queries. Use WebFetch to read full pages."
- Firecrawl (b): "Use Firecrawl search for discovery. Use Firecrawl scrape for specific URLs. Use Firecrawl crawl to map entire sites. Fall back to WebFetch if Firecrawl errors."
- Linkup (c): "Use Linkup search for structured results. Fall back to WebFetch for direct page access."
- Tavily (d): "Use Tavily search for AI-optimized results. Fall back to WebFetch for direct page access."
- Multiple (e): combine instructions for all selected tools

### File 10: `.claude/agents/dr-evaluator.md`

Project-local evaluator subagent. Copy the default evaluator from the installed agents.

IMPORTANT: The subagent frontmatter MUST use `disallowedTools` (denylist), NOT `tools` (allowlist). An allowlist blocks MCP tools from being inherited. Use this frontmatter:

```yaml
---
name: dr-evaluator
description: Evaluates research quality against binary criteria. Used by /dr-improve.
disallowedTools:
  - Write
  - Edit
model: sonnet
---
```

### File 11: `prompts/` directory

One .md file per session/phase. Each prompt file contains:
- Current phase and its objective
- Reference to todo.md for task list
- Loop instruction (execute tasks until phase complete or stop condition)
- Termination condition
- Summary instruction (write to CHANGELOG on completion)

### File 12: `.claude/agents/dr-adversary.md`

Project-local adversary subagent. Copy from the installed `agents/dr-adversary.md`.

IMPORTANT: The subagent frontmatter MUST use `disallowedTools` (denylist), NOT `tools` (allowlist). Use this frontmatter:

```yaml
---
name: dr-adversary
description: Adversarial verification agent. Attacks completed research claims by checking source support, searching for contradictions, and cross-referencing. Spawned by /dr-run.
disallowedTools:
  - Edit
model: sonnet
---
```

### Directory: `.research/source-cache/`

Create this directory. It will be populated by the researcher as it fetches URLs, and read by the adversary for cached source content.

## After Generation

Print the complete file tree and say:

"Your research project is scaffolded. Run /dr-run to start, or /dr-status to see the plan."
