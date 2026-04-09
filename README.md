<p align="center">
  <img src="docs/architecture.png" alt="deepresearch architecture" width="800">
</p>

<h1 align="center">deepresearch</h1>

<p align="center">
  <strong>Turn Claude Code into an autonomous research agent that runs for hours, not minutes.</strong>
</p>

<p align="center">
  <a href="#install">Install</a> &bull;
  <a href="#quick-start">Quick start</a> &bull;
  <a href="#how-it-works">How it works</a> &bull;
  <a href="#commands">Commands</a> &bull;
  <a href="#autonomous-mode">Autonomous mode</a> &bull;
  <a href="#self-improvement">Self-improvement</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/claude%20code-skills-blueviolet" alt="Claude Code Skills">
  <img src="https://img.shields.io/badge/agents-multi--agent-orange" alt="Multi-Agent">
</p>

---

## The problem

You prompt Claude Code with "research X." It does a surface-level pass in 20 minutes and stops. No structure, no verification, no memory across sessions.

**deepresearch** fixes this. It decomposes your research into 80-120 atomic tasks, spawns isolated subagents for each, verifies every output against schemas, and self-improves the researcher over time. One command to start. Walk away. Come back to structured, source-backed findings.

### What you get

- **Multi-session research** that picks up exactly where it left off
- **Structured output** with JSON schemas, source URLs, and verification for every claim
- **Self-improving researcher** that gets better each phase (eval, mutate, re-run, keep/revert)
- **Parallel execution** with automatic dependency analysis
- **Autonomous mode** for overnight unattended runs
- **Anti-hallucination rules** baked in: `NOT_FOUND` over guessing, sources required, conflicts flagged

---

## Install

```bash
git clone https://github.com/HadiFrt20/deepresearch.git ~/.claude/skills/deepresearch
cd ~/.claude/skills/deepresearch && ./setup
```

That's it. No dependencies. No build step. Open a new Claude Code session and the commands are available.

## Quick start

```
/dr-new     → interactive setup: 11 questions, full project scaffold
/dr-run     → execute tasks autonomously
/dr-status  → progress dashboard
```

`/dr-new` interviews you about your research goal, scope, and preferences, then generates 11 files: execution protocol, 80-120 tasks, schemas, eval criteria, and project-local subagents. `/dr-run` takes it from there.

---

## How it works

### Architecture

deepresearch has four layers that work together:

**User Commands** — Seven skills (`/dr-new`, `/dr-run`, `/dr-status`, `/dr-review`, `/dr-resume`, `/dr-improve`, `/dr-report`) drive the workflow from project setup to final deliverable.

**Orchestration** — The execution engine reads `CLAUDE.md` and `todo.md`, spawns subagents for each task, verifies output against JSON schemas, and logs results to `CHANGELOG.md`. A mode router selects sequential (one task at a time) or parallel (batches of 5 with dependency analysis).

**Subagents** — Three specialized agents, each with a focused role:
- **dr-researcher** — executes a single research micro-task with structured output and schema validation
- **dr-planner** — analyzes task dependencies for parallel safety, classifies tasks as SAFE/RISKY/BLOCKING
- **dr-evaluator** — scores data against binary eval criteria and identifies failure patterns

**Project Files** — `CLAUDE.md`, `todo.md`, `.research/ROADMAP.md`, `.research/CHANGELOG.md`, `.research/ARCHITECTURE.md`, `.research/evals.md`, and `data/*.json` provide persistent state across sessions. Research tools (WebSearch, WebFetch, or MCP servers like Firecrawl, Linkup, Tavily) are configured per-project.

### Two ratchets

The system runs two interlocking loops:

```
Research ratchet (/dr-run)              Improvement ratchet (/dr-improve)
─────────────────────────               ─────────────────────────────────
find task → spawn subagent →            score data → analyze failures →
verify output → mark done/fail →        mutate researcher instructions →
log → next task                         re-run sample → keep if better
```

The researcher subagent's prompt is the **trainable parameter**. Eval pass rate is the **metric**. Binary eval criteria, no vibes.

### Generated project structure

```
your-project/
├── CLAUDE.md                     # execution protocol + rules
├── todo.md                       # 80-120 atomic tasks with search strategies
├── .research/
│   ├── PROJECT.md                # research brief
│   ├── ARCHITECTURE.md           # JSON schemas + methodology ADRs
│   ├── ROADMAP.md                # 4-6 phased plan with status tracking
│   ├── CHANGELOG.md              # append-only execution log
│   ├── evals.md                  # binary quality criteria
│   └── eval-history/             # researcher version history + scores
├── .claude/agents/
│   ├── dr-researcher.md          # project-local researcher (the trainable part)
│   ├── dr-evaluator.md           # quality evaluator
│   └── dr-planner.md             # dependency analyzer for parallel mode
├── prompts/                      # per-session/phase prompt files
├── data/                         # structured JSON research output
└── output/
    └── final-report.md           # synthesized deliverable
```

---

## Commands

| Command | What it does |
|---------|-------------|
| `/dr-new` | Interactive setup (11 questions) then full project scaffold |
| `/dr-run` | Autonomous task execution with verification |
| `/dr-status` | Dashboard: tasks, phases, data completeness |
| `/dr-review` | Audit data quality against schemas |
| `/dr-resume` | Pick up where you left off after a break |
| `/dr-improve` | Self-improvement loop on the researcher |
| `/dr-report` | Synthesize findings into final deliverable |

---

## Execution modes

Control how tasks run. Set the default during `/dr-new` or override per-run:

| Mode | Command | Best for |
|------|---------|----------|
| Sequential | `/dr-run sequential` | First runs, debugging |
| Sequential + auto-improve | `/dr-run sequential-auto-improve` | Quality-focused research |
| Parallel (batches of 5) | `/dr-run parallel` | Speed, large task sets |
| Parallel + auto-improve | `/dr-run parallel-auto-improve` | Overnight runs |

**Parallel mode safety:** Before the first parallel run, a `dr-planner` subagent analyzes task dependencies and classifies each as SAFE, RISKY, BLOCKING, or CROSS-PHASE. Race conditions on shared files are handled with temp files and atomic merges.

---

## Autonomous mode

The default permission mode is set during `/dr-new` (question A9). In "ask" mode, `/dr-run` requests approval on every web search, fetch, and file write. Safe, but blocks overnight runs.

```
/dr-run auto                  # fully autonomous, no prompts
/dr-run parallel auto         # parallel + autonomous
```

**Auto mode is independent of execution mode** — it controls permissions (ask vs don't ask), not strategy (sequential vs parallel). Combine freely.

Safety rails still apply in auto mode:
- Source URLs required for every claim
- `NOT_FOUND` over guessing (never invents data)
- 10-minute time budget per task
- 3 consecutive failures stops the run (parallel mode falls back to sequential first)
- 3-hour cumulative runtime stops the run
- Rate-limit detection triggers checkpoints
- `Ctrl+C` to interrupt at any time

Set auto as the project default during `/dr-new` (question A9), or override per-run with `/dr-run auto` or `/dr-run ask`.

---

## Tool support

During `/dr-new`, pick your research tools. The choice auto-configures the researcher subagent and CLAUDE.md.

| Tool | What it adds |
|------|-------------|
| **Native** (WebSearch + WebFetch) | Works out of the box, no setup |
| **Firecrawl MCP** | Search, scrape, crawl, site mapping |
| **Linkup MCP** | Search with structured extraction |
| **Tavily MCP** | Search optimized for AI agents |
| **Multiple** | Combine any of the above |

Change tools later by editing the Tool priority section in your project's CLAUDE.md.

---

## Self-improvement

`/dr-improve` runs a ratchet loop on the researcher subagent:

1. **Score** current data against binary eval criteria
2. **Analyze** failure patterns (which criteria fail, why, where)
3. **Mutate** 1-3 concrete instructions in the researcher prompt
4. **Re-run** a sample of failed tasks with the new researcher
5. **Score again** and **keep if better, revert if not**

```
/dr-improve              # single improvement cycle
/dr-improve --cycles 3   # three rounds
/dr-improve --criteria   # add new eval criteria first
```

All versions are saved to `.research/eval-history/`. In auto-improve execution modes, this runs automatically between phases.

---

## Data quality guarantees

Every research entry follows strict rules:

| Rule | What it means |
|------|--------------|
| `NOT_FOUND` | Could not find this data point. Never guesses. |
| `UNVERIFIED: {value}` | Single-source claim, not cross-referenced |
| `CONFLICTING: {v1} vs {v2}` | Sources disagree, both values preserved |
| `INACCESSIBLE` | Website down or blocked |
| `sources[]` required | Every entry must cite at least 1 URL |
| `status` required | COMPLETE, INCOMPLETE, or INACCESSIBLE |

---

## Inspired by

| Project | What we took |
|---------|-------------|
| [GSD](https://github.com/zackiles/claude-code-gsd) | Spec-driven scaffolding (interview, specs, tasks, execute, verify) |
| [gstack](https://github.com/k響/gstack) | Role-based skills (one command per job, opinionated defaults) |
| [Karpathy's autoresearch](https://x.com/karpathy) | The ratchet loop (run, score, keep/revert, repeat) |
| Self-improving skills | Eval, mutate, re-run, keep if better |

---

## Update

```bash
cd ~/.claude/skills/deepresearch && git pull && ./setup
```

## Uninstall

```bash
cd ~/.claude/skills/deepresearch && ./uninstall
rm -rf ~/.claude/skills/deepresearch  # optional: remove the repo
```

## Requirements

- Claude Code 2.1.88+
- Max plan recommended for auto mode and long sessions
- Firecrawl MCP recommended but not required

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). No build step. Edit SKILL.md files, save, test.

## License

MIT
