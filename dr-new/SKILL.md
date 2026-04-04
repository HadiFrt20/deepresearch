---
name: dr-new
description: Set up a new autonomous research project. Interviews you with 7 questions, then generates the full project scaffold (specs, roadmap, tasks, schemas, execution protocol). Use when starting any research project.
---

You are setting up an autonomous multi-session research project. Interview the user, then generate every file needed to run the research autonomously for hours.

## PHASE A: Interview

Ask one question at a time. Wait for the answer before asking the next.

**A1 — Mission:** What are you researching? What specific question are you trying to answer?

**A2 — Decision:** What will you do with this research? (build a product, write a post, pitch investors, make a career move, other)

**A3 — Done condition:** What artifact do you want at the end? (strategic brief, spreadsheet, landscape map, opportunity scorecard, other)

**A4 — Scope:** How many categories/segments? Geographic scope? Company stage filter? Depth per entity (surface vs deep)?

**A5 — Constraints:** How many overnight sessions (1-5)? Quality vs breadth? Budget ceiling for API credits?

**A6 — Methodology:** Breadth-first or depth-first? Single-pass or cross-referenced? How to handle dead ends (skip, retry, flag)?

**A7 — Tools:** What research tools do you have available?
  a) Native only (WebSearch + WebFetch built into Claude Code — no setup needed)
  b) Firecrawl MCP (search, scrape, crawl, map — best for deep site extraction)
  c) Linkup MCP (search with structured extraction)
  d) Tavily MCP (search optimized for AI agents)
  e) Multiple — list which ones

Tell the user: "If you're not sure, pick (a). You can always add tools later by editing the Tool priority section in CLAUDE.md."

## PHASE B: Generate Project Scaffold

After all 7 answers, generate these files without asking more questions:

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
- `"UNVERIFIED: [value]"` — single-source fact, not cross-referenced
- `"CONFLICTING: [v1] vs [v2]"` — sources disagree

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

Binary evaluation criteria for research quality (used by /dr:improve).

Default criteria:
1. Does the entry have all required schema fields populated?
2. Does the entry have at least 1 source URL?
3. Is the status field set to a valid value (COMPLETE, INCOMPLETE, or INACCESSIBLE)?
4. Are funding/date claims backed by a source?
5. Is the entity's own website listed in sources?

Add domain-specific criteria based on the research topic.

### File 6: `.research/verify.sh`

Bash script that:
- Counts completed `[x]`, failed `[!]`, and remaining `[ ]` tasks in todo.md
- Lists data files with entry counts and field completeness percentages
- Shows last 10 changelog entries

Make it executable.

### File 7: `CLAUDE.md`

Execution protocol containing:

**Mission** — 1 paragraph from PROJECT.md

**Bootstrap** — "Read these files first: .research/ROADMAP.md, todo.md, .research/CHANGELOG.md"

**Execution protocol:**
1. Find first unchecked task in todo.md
2. Spawn dr-researcher subagent with task description, output schema, output file path
3. Verify output: file exists, valid JSON/markdown, required fields populated, sources array non-empty, status field set
4. Pass → mark `[x]` in todo.md; Fail → mark `[!]` in todo.md
5. Append to CHANGELOG: `[ISO timestamp] | [task_id] | DONE/FAIL | [file] | [summary]`
6. Every 5 tasks: write checkpoint to CHANGELOG (completed/failed/remaining counts, patterns, continue/stop recommendation)
7. Repeat from step 1

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

**Self-improvement:** After each phase completes, run /dr:improve to evaluate and refine the researcher subagent.

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
description: Executes a single research micro-task with structured output. Spawned by /dr:run.
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
description: Evaluates research quality against binary criteria. Used by /dr:improve.
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

## After Generation

Print the complete file tree and say:

"Your research project is scaffolded. Run /dr:run to start, or /dr:status to see the plan."
