# ECC-Guide
A straightforward guide to Everything Claude Code

# ECC Setup Guide — for an AI/ML Engineer using Claude Code

A complete, reusable setup for working on Python / ML projects (including LLM apps) with
Anthropic's Claude Code, using the ECC (Everything Claude Code) harness by affaan-m.

Everything in this guide is taken from the actual ECC repo `SKILL.md` / command files,
not inferred. Where a name or behavior matters, it has been checked against the source.

> **Reality check before you start.** ECC ships 261 skills, 64 agents, and 84 commands.
> You do **not** want all of it — loading everything bloats your context window, which is
> the exact problem ECC's own guides warn about. This guide installs a deliberately small,
> reusable core (Parts 1–9) and lists the ML/AI-relevant project-specific skills separately
> (Part 5b) so they stay opt-in. After installing, run `/plugin list ecc@ecc` to see what
> actually landed, since names and behavior drift between versions.
>
> **On completeness:** the skill catalog was reviewed via ECC's full categorized skills
> reference plus the individual `SKILL.md` files for the entries that matter to ML/AI work —
> not by opening all 261 files individually (most are framework/language packs irrelevant to
> Python/ML). Entries marked ~ in Part 5b are known from the catalog but not individually
> verified; check their `SKILL.md` before relying on them.

---

## Part 0 — Mental model: the four component types

| Type | Lives in | How it activates |
|---|---|---|
| **Rules** | `~/.claude/rules/ecc/` | Always loaded, every session. Cannot be shipped via plugin — copy manually |
| **Skills** | `~/.claude/skills/` | Primary workflow surface. Invoked by name, auto-suggested, or reused by agents |
| **Commands** | `~/.claude/commands/` (or `/ecc:*` via plugin) | Legacy slash shims. Being migrated into skills |
| **Hooks** | `~/.claude/hooks/` + `settings.json` | Fire automatically on tool/lifecycle events |
| **Agents** | `~/.claude/agents/` | Subagents the orchestrator delegates scoped tasks to |

Two things ECC does **not** do: it does not save your code (that's git), and it does not
automatically know your codebase (you generate that once with `codebase-onboarding`).

---

## Part 1 — Install (one path only, never stack)

```bash
# 1. Install the plugin
/plugin marketplace add https://github.com/affaan-m/ECC
/plugin install ecc@ecc

# 2. Clone the source separately, only to grab rules + the skills that need manual copy
git clone https://github.com/affaan-m/ECC.git ~/ecc-source
```

> Do **not** also run `./install.sh --profile full` after a plugin install. That copies the
> same surfaces into your user dirs and creates duplicate skills + duplicate hook behavior.
> Pick the plugin path OR the manual installer, not both.

---

## Part 2 — Rules (manual copy, always required)

The plugin system cannot distribute rules. Copy the common set plus Python:

```bash
mkdir -p ~/.claude/rules/ecc
cp -r ~/ecc-source/rules/common  ~/.claude/rules/ecc/
cp -r ~/ecc-source/rules/python  ~/.claude/rules/ecc/
```

- `rules/common` — 8 files: coding style, git workflow, testing, security, performance,
  patterns, hooks, agents (language-agnostic principles).
- `rules/python` — Python idioms, pytest, ruff/black formatting.

Skip the other language packs (typescript, golang, swift, php, arkts, etc.) unless you use them.

---

## Part 3 — Memory & context (the part you most care about)

There are **three** distinct mechanisms. Use all three; they do different jobs.

### 3a. `ck` (Context Keeper) — per-project cross-session memory

Persistent per-project memory. Auto-loads project context on session start, tracks sessions
with git activity, writes to native memory via deterministic Node.js scripts.

```bash
# Copy the WHOLE directory (it has hooks + scripts, not just SKILL.md)
cp -r ~/ecc-source/skills/ck ~/.claude/skills/ck
```

Register its SessionStart hook in `~/.claude/settings.json` (full file in Part 8):

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command",
        "command": "node \"$HOME/.claude/skills/ck/hooks/session-start.mjs\"" }] }
    ]
  }
}
```

The hook injects ~100 tokens per session (a compact 5-line summary) and detects unsaved
sessions, git activity since last save, and goal mismatches vs CLAUDE.md.

**How it separates projects:** data lives at `~/.claude/ck/`. A `projects.json` index maps
each project's filesystem **path** → `{name, contextDir}`. Each project gets its own
`contexts/<name>/context.json`. Lookup is path-based — it matches your current working
directory against the index.

Commands:

| Command | Purpose |
|---|---|
| `/ck init` | Register the current project (auto-detects name, stack, goal; you confirm) |
| `/ck save` | Save session state: summary, where you left off, next steps, decisions, blockers |
| `/ck resume [name]` | Full briefing — run at the **start** of every session |
| `/ck list` | All registered projects |
| `/ck info [name]` | Quick snapshot, no follow-up |
| `/ck forget [name]` | Remove a project (asks to confirm) |

Never hand-edit `context.json` or `CONTEXT.md` — always go through the commands.

> **Known caveat:** there's a reported issue where the auto-injection on SessionStart doesn't
> reliably push content into context. Treat `/ck resume` as a deliberate manual habit at the
> start of each session rather than assuming it happens invisibly.

### 3b. ECC hooks-runtime — automatic session summaries

ECC's own SessionStart / Stop / PreCompact hooks that persist session summaries under
`~/.claude/session-data/`. Install via the installer so paths are rewritten correctly:

```bash
# macOS / Linux
bash ~/ecc-source/install.sh --target claude --modules hooks-runtime
# Windows PowerShell
# pwsh -File ~/ecc-source/install.ps1 --target claude --modules hooks-runtime
```

Do **not** paste the repo's raw `hooks/hooks.json` into `settings.json` — it's
plugin-oriented and the installer rewrites command paths against your real Claude root.

### 3c. `continuous-learning-v2` — learns your patterns over time

Uses hooks for observation (100% reliable) and "instincts" as atomic units of learned
behavior with confidence scoring, project-scoped. (v1 is **deprecated** — don't install it.)

```bash
# Copy the WHOLE directory — has hooks, scripts, config.json
cp -r ~/ecc-source/skills/continuous-learning-v2 ~/.claude/skills/continuous-learning-v2
```

| Command | Purpose |
|---|---|
| `/instinct-status` | View learned instincts + confidence scores |
| `/instinct-export` | Export instincts (share / back up) |
| `/instinct-import <file>` | Import instincts |
| `/evolve` | Cluster related instincts into full skills |

---

## Part 4 — Understanding a codebase without re-exploring it every session

This is two separate tools — both belong in a reusable setup.

### 4a. `codebase-onboarding` skill — run ONCE per project

Analyzes an unfamiliar codebase and generates a structured onboarding guide (architecture
map, entry points, conventions) **and a starter project-level CLAUDE.md**. Four phases:
reconnaissance → architecture mapping → convention detection → generate artifacts.

Invoke conversationally:
- "onboard me to this codebase"
- "generate a CLAUDE.md for this project"
- "update the CLAUDE.md with current project conventions"

If a `CLAUDE.md` already exists it enhances rather than replaces it, and flags what changed.
This is the tool that actually creates your **project-level** CLAUDE.md (see Part 7 for the
distinction from the global one).

### 4b. `/ecc:update-codemaps` — run periodically as the code grows

Generates token-lean architecture docs in `docs/CODEMAPS/` (project type, source dirs,
entry points). In future sessions Claude loads these maps instead of re-reading source —
far cheaper on tokens than exploring fresh each time. Re-run after significant structural changes.

---

## Part 5 — Per-feature workflow

A repeatable sequence for every non-trivial feature. Follow the sections in order.

---

### Session start

**First time in this project** (run once per project, not per session):

```bash
"onboard me to this codebase"    # codebase-onboarding → architecture guide + project CLAUDE.md
/ck init                         # register project in Context Keeper
/ecc:update-codemaps             # generate docs/CODEMAPS/ for token-cheap future navigation
```

**Every subsequent session** (run at the top of each session):

```bash
/ck resume <project-name>        # loads last summary, next steps, open decisions
```

**When to save and maintain context:**

| Action | When |
|---|---|
| `/ck save` | Before ending every session — saves summary, where you left off, decisions, blockers |
| `/ecc:update-codemaps` | After significant structural changes — new modules, renamed dirs, new entry points |

---

### 1. Research

Run before planning when you're unsure of the best approach or need to find existing solutions. Skip for work you already understand well.

| Skill | What it does | When to use | Requirements |
|---|---|---|---|
| `search-first` | Checks your repo first, then PyPI/npm, then GitHub, then MCP catalog — finds existing solutions before you write anything custom | Before building any utility or adding a dependency; when you're not sure if the solution already exists somewhere | `gh` CLI |
| `documentation-lookup` | Fetches live library and API docs via Context7 MCP instead of relying on potentially stale training data | "How do I use library X or API Y?" — especially for packages with fast-moving APIs or version-specific behaviour | Context7 MCP configured |
| `deep-research` | Multi-source web synthesis with citations via Firecrawl or Exa. Returns a structured report with sourced findings. | "What's the best approach to solve problem Z?" — architecture tradeoffs, technology evaluation, unfamiliar domains | Firecrawl or Exa MCP configured |

**Suggested order when you need all three:**
```
search-first          # does something already solve this?
documentation-lookup  # how exactly does the relevant library/API work?
deep-research         # what's the best overall approach?
```

---

### 2. Planning

Always plan before implementing non-trivial work. Pick the path based on how clear the scope is.

**Path A — scope is clear, work is non-trivial**

"Scope is clear" means you know *what* needs to happen and there's nothing to debate — not that the implementation is simple. `/plan` still earns its keep by scanning the codebase for conventions to mirror and surfacing risks before you touch code.

```bash
/ecc:plan "description"              # scans codebase → lists risks → waits for your confirmation
```

Use when: bug fix, scoped refactor, known migration, extending an existing pattern — anything where *what* is settled but the work touches multiple files or has real complexity.

**Path B — scope is unclear or stakeholders need to align**

```bash
/ecc:plan-prd "the idea"             # requirements doc → .claude/prds/*.prd.md
                                     # captures: problem, evidence, users, hypothesis, MVP, out-of-scope
# review and edit the PRD; align on what before anyone codes how
/ecc:plan .claude/prds/name.prd.md   # implementation plan → .claude/plans/*.plan.md
```

Use when: scope is ambiguous, multiple stakeholders involved, or the feature is large enough that writing down *why* is cheaper than relitigating scope mid-build.

---

### 3. Implementation

```bash
tdd-workflow                         # RED → GREEN → REFACTOR
```

| What it enforces | Detail |
|---|---|
| Tests before code | Write failing tests first, implement minimally to pass |
| Git checkpoints | Commit at RED, at GREEN, after refactor — clean rollback points at each stage |
| 80% coverage | Unit + integration + E2E |

Skipping `tdd-workflow` is fine if you write tests naturally alongside code. The skill adds value as an enforced guardrail — it doesn't add capability you don't already have.

---

### 4. Testing & review

Run after implementation, before opening a PR:

| Skill / command | What it does | When |
|---|---|---|
| `python-testing` | Reference skill for pytest patterns — fixture design, mocking, parametrization, coverage setup. Use **before** `tdd-workflow` if you're unsure how to structure tests for the feature; gives you the patterns to follow. Does not execute TDD itself. | When test structure is unclear — e.g. how to mock an ONNX model, parametrize phoneme inputs, or structure fixtures for DB-backed tokenizer tests |
| `/ecc:code-review` | Delegates to `code-reviewer` agent — quality, maintainability, patterns | After every implementation |
| `security-review` | OWASP-style checklist — secrets, injection, auth, input validation | Before any external-facing code ships |
| `/ecc:quality-gate` | Verification gate: tests green, coverage ≥ 80%, lint clean | Final check before PR |

---

### 5. Debugging

| Skill / command | What it does | When |
|---|---|---|
| `/ecc:build-fix` | Delegates to `build-error-resolver` — diagnoses and fixes build and type errors | On any build failure, before manual debugging |
| `error-handling` | Reviews code for swallowed errors, missing propagation, and silent failures | When bugs are hard to reproduce or errors aren't surfacing cleanly |

---

### 6. Ship

```bash
/ecc:pr          # validates branch state, generates PR title/body from commits,
                 # auto-links .claude/prds/ and .claude/plans/ artifacts in the PR body
/ck save         # save session context before stopping
```

---

Skip planning entirely only for trivial single-file changes with no downstream effects.
Heuristic: if you'd naturally spend 2+ minutes thinking before coding, run at least Path A.

---

## Part 5b — ML/AI project-specific skills (NOT for every project)

These are the skills worth knowing about as an ML/AI engineer. None of them belong in your
always-on global setup — install or invoke them only on the projects that need them, because
each one consumes context. Grouped by what kind of project triggers them.

Names verified against the repo's `SKILL.md` files are marked ✓. Names known only from the
catalog / release notes (not individually opened) are marked ~ — confirm the exact behavior
with `/plugin list ecc@ecc` and the skill's own `SKILL.md` after install.

### Core ML/LLM engineering workflow

| Skill | ✓/~ | What it does | Use when |
|---|---|---|---|
| `mle-workflow` | ✓ | Frames ML work as explicit contracts: data + metric contract (entity grain, label timing, label confidence, point-in-time joins, split policy, dataset snapshot), eval harness, serving path, monitoring, docs. Routes LLM/embedding workloads by quality/latency/budget. Packages changes for review with reproducible test evidence | Any production ML/MLE change |
| `ai-first-engineering` / `agentic-engineering` | ✓ | Eval-first execution: define capability + regression evals, capture baseline failure signatures, implement, re-run, compare deltas, check regressions. Decompose into agent-sized units, route model tiers by complexity | When agents do most of the implementation and you enforce quality with evals |
| `cost-aware-llm-pipeline` | ✓ | Model routing, budget tracking, caching for the LLM app **you ship** (cheap model for intent/retrieval, strong only for final generation) | Building any LLM-powered feature |
| `regex-vs-llm-structured-text` | ✓ | Decision framework: when to parse text with regex vs an LLM call | Any pipeline extracting structure from text |
| `content-hash-cache-pattern` | ✓ | SHA-256 content-hash caching for file processing | Ingestion/embedding pipelines |

### LLM app security (any agent with a tool/action surface)

| Skill | ✓/~ | What it does |
|---|---|---|
| `llm-trading-agent-security` | ✓ | Prompt-injection defense for execution-capable agents: input sanitization, layered controls (prompt hygiene, spend policy, simulation, execution limits, isolation). The patterns apply to any LLM agent that can take actions, not just trading |
| `security-scan` | ✓ | AgentShield integration — scans CLAUDE.md, settings, MCP configs, hooks, agents, skills for secrets, permission issues, hook injection, MCP risk. Run before trusting an autonomous setup |

### Data & retrieval (RAG, analytics, pipelines)

| Skill | ✓/~ | What it does | Use when |
|---|---|---|---|
| `clickhouse-io` | ✓ | ClickHouse analytics, column-store queries, aggregations | Analytics / event data |
| `postgres-patterns` | ✓ | PostgreSQL index strategies, query plans, JSONB | Postgres / pgvector RAG stores |
| `database-migrations` | ✓ | Migration patterns (Prisma, Drizzle, Django, Go) | Evolving a schema / metadata store |
| `nutrient-document-processing` | ~ | Document processing via Nutrient API | RAG ingestion of PDFs/docs |
| `videodb` | ~ | Video/audio: ingest, search, edit, generate, stream | Multimodal / media ML |

### Serving & MLOps

| Skill | ✓/~ | What it does |
|---|---|---|
| `api-design` | ✓ | REST design, pagination, versioning, error responses, rate limiting — for the endpoint you serve the model behind |
| `docker-patterns` | ✓ | Docker Compose, networking, volumes, container security |
| `deployment-patterns` | ✓ | CI/CD, health checks, rollbacks, blue-green |
| `mcp-server-patterns` | ~ | Patterns for building MCP servers — if you expose tools/data to LLMs |
| `documentation-lookup` | ~ | API reference research workflow |

### Performance / optimization pack (newer additions — verify SKILL.md)

Turn repeated speed/throughput prompts into bounded, benchmarked workflows. Relevant to training
throughput, inference latency, and pipeline tuning.

| Skill | ✓/~ | What it does |
|---|---|---|
| `benchmark-optimization-loop` | ~ | Bounded benchmark-driven optimization loop |
| `data-throughput-accelerator` | ~ | Data pipeline throughput tuning |
| `latency-critical-systems` | ~ | Latency-critical system patterns (inference paths) |
| `parallel-execution-optimizer` | ~ | Parallel execution tuning |
| `recursive-decision-ledger` | ~ | Bounded recursion / decision-ledger workflow |

### Deep learning only

| Component | ✓/~ | What it does |
|---|---|---|
| `pytorch-patterns` (skill) | ✓ | Deep-learning workflow patterns |
| `pytorch-build-resolver` (agent) | ✓ | PyTorch/CUDA build + training error resolution |

Skip both entirely if you build on API-based models rather than training your own.

### Autonomous / batch execution

| Skill | ✓/~ | What it does |
|---|---|---|
| `autonomous-loops` | ✓ | Six loop architectures: sequential pipeline (`claude -p` chains), NanoClaw REPL, infinite agentic loop (decomposer + parallel workers), continuous PR loop with CI gates, DAG orchestration, plus a "de-sloppify" cleanup pass. Useful for unattended eval sweeps or batch jobs |

### Relevant agents (delegated, not invoked directly)

| Agent | ✓/~ | Role |
|---|---|---|
| `mle-reviewer` | ✓ | Reviews production ML changes — pipeline, eval, serving, monitoring |
| `python-reviewer` | ✓ | Python code review (`/python-review`) |
| `database-reviewer` | ✓ | Auto-delegated DB/query review |
| `docs-lookup` | ✓ | Documentation/API lookup |
| `architect` | ✓ | System design (RAG vs fine-tune, store choice) |
| `security-reviewer` | ✓ | Vulnerability + secret scanning before shipping |

### How to install just these per-project

Use the component-scoped installer rather than copying the whole skills dir:

```bash
npx ecc consult "mlops training model deployment" --target claude   # preview matches
npx ecc install --profile minimal --target claude --with capability:machine-learning
```

Or copy individual skills into a **project-level** dir so they don't pollute global context:

```bash
mkdir -p .claude/skills
cp -r ~/ecc-source/skills/mle-workflow .claude/skills/
cp -r ~/ecc-source/skills/cost-aware-llm-pipeline .claude/skills/
```

Project-level skills (`.claude/skills/`) override and supplement user-level ones, and only load
for that project — exactly what you want for these.

---

## Part 6 — Always-on hooks (set once)

Beyond the memory hooks in Part 3, add these. They fire automatically — you never invoke them.

### 6a. `strategic-compact` — suggests `/compact` at logical points

Auto-compaction triggers at arbitrary points, often mid-task. This suggests manual compaction
at phase boundaries instead.

```bash
cp -r ~/ecc-source/skills/strategic-compact ~/.claude/skills/strategic-compact
```

Decision table (when to actually compact):

| Transition | Compact? | Why |
|---|---|---|
| Research → Planning | Yes | Research is bulky; the plan is the distilled output |
| Planning → Implementation | Yes | Plan is saved in a file; free up context for code |
| Implementation → Testing | Maybe | Keep if tests reference recent code |
| Debugging → next feature | Yes | Debug traces pollute unrelated work |
| Mid-implementation | **No** | You lose variable names, file paths, partial state |
| After a failed approach | Yes | Clear the dead-end reasoning |

What survives compaction: CLAUDE.md, TodoWrite list, memory files, git state, files on disk.
What's lost: intermediate reasoning, file contents you read, tool-call history.

### 6b. Python formatting/typing hooks

On every `.py` edit, auto-run ruff + black + mypy so you never accumulate lint debt.

(Both 6a and 6b are wired into the settings.json in Part 8.)

---

## Part 7 — CLAUDE.md: two levels, don't confuse them

| File | Who creates it | Scope |
|---|---|---|
| `~/.claude/CLAUDE.md` | **You, manually** (template below) | Global — every project |
| `<repo>/.claude/CLAUDE.md` | `codebase-onboarding` skill (Part 4a) | This project only |

Neither is created by `/ck init` — that only writes ck's own `context.json`.

### Global `~/.claude/CLAUDE.md` template

```markdown
## Model routing
- Haiku: exploration, file search, reading docs, single-file edits
- Sonnet: default for all coding (~90% of tasks)
- Opus: first attempt failed, task spans 5+ files, architecture, security-critical code

## Code standards (Python)
- ruff + black + mypy on every file
- No print/console debug statements in committed code
- TDD: tests before implementation, 80% coverage minimum

## Context discipline
- Keep under 10 MCPs enabled, under 80 tools active
- /compact after research->plan and plan->implement transitions
- /ck save before ending any session

## Delegation
- code-reviewer after every implementation
- security-reviewer before any external-facing code ships
- build-error-resolver on build failure before manual debugging
```

---

## Part 8 — Full `~/.claude/settings.json`

Merge of the SessionStart (ck), strategic-compact (PreToolUse), and Python quality (PostToolUse) hooks:

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command",
        "command": "node \"$HOME/.claude/skills/ck/hooks/session-start.mjs\"" }] }
    ],
    "PreToolUse": [
      { "matcher": "Edit",
        "hooks": [{ "type": "command",
          "command": "node $HOME/.claude/scripts/hooks/suggest-compact.js" }] },
      { "matcher": "Write",
        "hooks": [{ "type": "command",
          "command": "node $HOME/.claude/scripts/hooks/suggest-compact.js" }] }
    ],
    "PostToolUse": [
      { "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\.py$\"",
        "hooks": [{ "type": "command",
          "command": "ruff check --fix \"$file_path\" && black \"$file_path\"" }] },
      { "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\.py$\"",
        "hooks": [{ "type": "command",
          "command": "mypy \"$file_path\" --ignore-missing-imports" }] }
    ]
  }
}
```

> If you installed the hooks-runtime module (Part 3b) via the installer, check that its
> SessionStart/Stop/PreCompact entries don't collide with the ones above — the installer
> writes to `~/.claude/hooks/hooks.json`, which Claude Code v2.1+ auto-loads. Don't duplicate
> the same hook in both `settings.json` and `hooks.json`.

---

## Part 10 — What saves what (so you're never surprised)

| Mechanism | Saves | Does NOT save |
|---|---|---|
| Editor autosave | Your code, to disk, immediately | — |
| `git commit` | Code history / changes | — |
| `/ck save` | Session context: summary, next steps, decisions, blockers | Code |
| hooks-runtime Stop hook | Automatic session summary | Code |
| `continuous-learning-v2` | Reusable patterns/instincts | Code |
| `codebase-onboarding` | Project CLAUDE.md + onboarding guide | Code |
| `/ecc:update-codemaps` | Architecture maps in docs/CODEMAPS/ | Code |

ECC's memory answers "what were we doing and where did we leave off." Git answers "what code
exists and what changed." They're complementary.

---

## Appendix — Reusable core, minimal list

If you ignore everything else, this is the durable set worth having in every project:

- **Rules:** common + python
- **Memory:** ck (+ SessionStart hook), hooks-runtime, continuous-learning-v2
- **Context:** strategic-compact, codebase-onboarding, /ecc:update-codemaps
- **Per-feature:** search-first -> /ecc:plan -> tdd-workflow -> /ecc:code-review ->
  security-review -> /ecc:quality-gate
- **ML/LLM (project-specific, see Part 5b):** mle-workflow, ai-first-engineering,
  cost-aware-llm-pipeline, regex-vs-llm-structured-text, llm-trading-agent-security; plus
  data/serving/perf skills as the project needs them
- **Global CLAUDE.md:** model-routing + standards + context discipline

A final caveat: ECC is one experienced developer's opinionated system, MIT-licensed and
moving fast (weekly releases). Names and behaviors shift between versions. After install,
`/plugin list ecc@ecc` is the source of truth for what you actually have, and each skill's
`SKILL.md` is the source of truth for what it does.
