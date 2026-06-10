# ECC Setup Guide for Cursor — AI/ML Engineer Edition

A reusable ECC (Everything Claude Code) setup for Python / ML projects in **Cursor IDE**, set up so
the memory and context are **shared with Claude Code** and you can switch between the two tools on the
same project. This is the Cursor counterpart to the Claude Code guide — most concepts are identical;
the differences are install method, config format, and a few invocation details.

Verified against ECC's Cursor integration docs, ECC's issue tracker, Cursor's current rules/hooks
documentation, and a **fresh-machine dry run** (ECC 2.0.0, June 2026 — clone → install → doctor → hook
path check). Where Cursor genuinely differs, it's called out — and where earlier claims were outdated
(notably hooks and install CLI invocation), that's corrected inline.

> **The single most important thing to understand:** Cursor is not Claude Code. ECC was built
> Claude-Code-first and *translated* to Cursor. Some pieces port cleanly (rules, agents, skills,
> and — as of current Cursor versions — hooks), and some don't exist on Cursor at all (the plugin
> marketplace, the `/ecc:*` slash-command namespace). Don't assume a feature works identically, but
> note that Cursor's native hooks support has improved a lot recently (see Part 5).

---

## Part 0 — How ECC maps onto Cursor (the translation layer)

ECC ships a `.cursor/` directory containing pre-translated versions of its components. The
installer writes adapted copies into your project's `.cursor/` folder. The mapping:

| ECC component | Claude Code | Cursor equivalent | Port quality |
|---|---|---|---|
| **Rules** | hierarchical `~/.claude/rules/` | `.cursor/rules/*.mdc`, flattened, with YAML frontmatter + globs | ✓ Full |
| **Agents** | `~/.claude/agents/*.md`, short model IDs | `.cursor/agents/*.md`, full model IDs, tools → `alwaysAllow` | ✓ Full |
| **Skills** | `~/.claude/skills/` | `.cursor/skills/`, near-identical markdown | ✓ Full |
| **Commands** | `/ecc:*` slash commands | path-updated; multi-agent ones stubbed | ✓ Partial |
| **Hooks** | 8 native event types | **native Cursor hooks** (`preToolUse`, `preCompact`, `sessionStart`, etc.) — ECC registers them in Cursor's Hooks settings | ✓ Mostly |
| **MCP config** | `.mcp.json` | translated env-var interpolation | ✓ Full |
| **Plugin marketplace** | `/plugin install` | **does not exist** — install is file-based only | ✗ None |

**Correction worth noting:** earlier ECC docs (and an earlier version of this guide) said Cursor
had no native hook lifecycle and everything was emulated via adapter scripts. That's now outdated —
current Cursor versions have a real **Hooks** settings panel, and ECC registers its hooks there as
native events. A confirmed install registers hook entries in `.cursor/hooks.json` (16+ entries across events like
`sessionStart`, `preCompact`, `afterFileEdit`, etc.). So the auto-format, compaction, and memory hooks
described in the Claude Code guide *can* fire on Cursor now. The remaining caveat is about whether
they're *effective*, not whether they're *registered* — see Part 5.

---

## Part 1 — Install (Cursor target)

There is **no plugin install** for Cursor. You use the selective installer, which is actually the
*primary* path for non-Claude harnesses.

```bash
# Clone the source
git clone https://github.com/affaan-m/ECC.git ~/ecc-source
cd ~/ecc-source
npm install
```

### Two ways to scope the install

**A) Broad, by language profile** — simplest, pulls the whole Python lane:

```bash
./install.sh --target cursor python
# Windows: .\install.ps1 --target cursor python
```

**B) Selective, by typed component** — the precise path. The verified syntax uses `--profile`
plus `--with` / `--without` flags with **typed prefixes**, not a bare `--modules` list.

> **Run install from `~/ecc-source`.** On a fresh machine, bare `ecc install` or `npx ecc` from an
> arbitrary project directory often resolves to a stale/global CLI and fails (`Unknown install profile:
> minimal`, `Unknown command: consult`). Always invoke the installer from the cloned ECC repo:

```bash
cd ~/ecc-source   # or pass the full path to install.sh from your project dir

./install.sh --target cursor --profile core \
  --with lang:python \
  --with capability:machine-learning \
  --with agent:security-reviewer \
  --without skill:continuous-learning
```

Use `--profile core` (not `minimal`) when you want hook-backed memory, format-on-edit, or
continuous-learning — `minimal` deliberately excludes the `hooks-runtime` module and does not copy
backend hook scripts. Alternative from a project directory:

```bash
npm exec --prefix ~/ecc-source ecc -- install --target cursor --profile core \
  --with lang:python --with capability:machine-learning --dry-run
```

The prefixes that are confirmed to work: `lang:`, `agent:`, `skill:`, and `capability:`. There is
also a `--modules` flag (used for things like `--modules hooks-runtime`), but its valid values come
from the install manifest, not from guessed category names.

> **Caution on module names.** Category labels you may have seen — "Framework & Language",
> "Workflow & Quality", "Database" — are the *interactive wizard's* prompt labels, **not** CLI
> module IDs. Don't pass them as `--modules framework-language,workflow-quality,...`; a selective
> install tends to **fail silently on unrecognized names** rather than error, so you'd think you
> installed something you didn't. Always discover the real names first (below).

### Discover the real component/module names for your version

```bash
cd ~/ecc-source

# The advisor returns valid components + the exact install command for a goal
npx ecc consult "mlops training model deployment" --target cursor
npx ecc consult "agentic workflow research apis" --target cursor

# Or read the manifest directly — this is the source of truth for module IDs
cat manifests/install-profiles.json
cat manifests/install-modules.json

# Preview the file plan before applying (run from ~/ecc-source or use npm exec --prefix)
./install.sh --target cursor --profile core \
  --with lang:python --with capability:machine-learning --dry-run
```

After any install, confirm what actually landed:

```bash
node ~/ecc-source/scripts/doctor.js                 # environment + drift check
cat .cursor/ecc-install-state.json 2>/dev/null    # lists active modules + timestamps

# If you need hooks, confirm backend scripts landed and are reachable
test -f .cursor/scripts/hooks/session-start.js && echo "hook scripts installed"
test -f scripts/hooks/session-start.js && echo "hook path wired" || echo "needs symlink — see Part 5"
```

Either path writes adapted agents, skills, commands, rules, and hook-adapter scripts into the
**project's** `.cursor/` directory. Unlike Claude Code (where most config is user-level in
`~/.claude/`), Cursor config is primarily **project-level** in `.cursor/`.

### Two install gotchas specific to Cursor (from ECC's own issue tracker)

1. **`.cursor/AGENTS.md` pollution.** Older ECC versions copied ECC's own `AGENTS.md` (which
   *describes ECC itself*) into your project's `.cursor/`. Because Cursor loads `AGENTS.md`
   contextually as "this is what the project is," your agent can start thinking it's working on
   the ECC plugin rather than your project. **After install, check whether `.cursor/AGENTS.md`
   exists; if it describes ECC rather than your project, delete it or replace it.** ECC 2.0.0
   selective installs may skip this file — still verify.

2. **`.cursor/agents/` namespace collisions.** The installer writes ~48 agent files into
   `.cursor/agents/` without an `ecc-` prefix, so they can silently collide with your own custom
   agents of the same name. If you have custom agents, back them up before installing, and check
   `ls .cursor/agents/` afterward.

---

## Part 2 — Rules (the part that ports cleanest)

Cursor's modern rule system is `.cursor/rules/*.mdc` files — markdown with YAML frontmatter. ECC's
installer flattens its hierarchical rules into this format and injects `globs`. Cursor has four
activation modes, and choosing the right one is what keeps your context budget healthy:

| Mode | Frontmatter | When it loads | Use for |
|---|---|---|---|
| **Always Apply** | `alwaysApply: true` | every single request | tech-stack declaration, critical conventions — keep under ~200 words |
| **Auto Attached** | `globs: ["**/*.py"]` | when you edit matching files | the workhorse — language/area-specific patterns |
| **Agent Requested** | `description: "..."` | when the agent decides it's relevant | situational guidance |
| **Manual** | `@rule-name` | only when you reference it | on-demand checklists |

After ECC installs its Python rules into `.cursor/rules/`, do this cleanup:

- Make sure your **always-apply** rule (tech stack + core standards) is short — every token loads
  on every request. The Cursor community rule of thumb is keep always-apply under 200 words; total
  always-apply content under ~2,000 tokens.
- Convert the bulky Python pattern rules to **Auto Attached** with `globs: ["**/*.py"]` so they
  only load when you're in Python files, not on every chat.

Example of a lean always-apply rule (`.cursor/rules/000-stack.mdc`):

```markdown
---
alwaysApply: true
---
This is a Python ML project. Use ruff + black + mypy. TDD: tests before code, 80% coverage.
Model routing handled by the agent configs. Keep functions small; prefer composition.
```

Commit `.cursor/rules/` to your repo — that's how the whole team gets the same behavior.

---

## Part 3 — Memory & context: sharing it between Cursor and Claude Code

On Claude Code, memory persistence rides on native SessionStart/Stop/PreCompact hooks. Current Cursor
versions have these hook events too (your install registers `sessionStart`, `preCompact`, etc.), so
the persistence machinery does fire on both. The interesting question for you is the opposite of what
the ECC docs assume: you want the **same memory readable by both tools** so you can switch between
them on the same project.

### 3a. Shared memory root (do this — it's what enables interchangeable use)

ECC's docs recommend *separating* the data roots (`ECC_AGENT_DATA_HOME="$HOME/.cursor/ecc"` for
Cursor) specifically to avoid the two tools overwriting each other's session files. That's the right
call if you treat them as independent. **You want the opposite**, so do NOT set a separate root —
point both tools at the same one:

```bash
# In your shell startup (~/.zshrc or ~/.bashrc): leave BOTH tools on the same root.
# Either rely on the default (~/.claude) for both, or set an explicit shared root:
export ECC_AGENT_DATA_HOME="$HOME/.ecc-shared"
```

If you set an explicit shared root, set it identically in the environment both Cursor and Claude
Code launch from. If you'd rather not touch the env var at all, just *don't* set the Cursor-specific
override — both default to `~/.claude` and you're done.

**Why this actually works — the key keying detail:** the two memory systems key on things that are
identical across tools, so the same project resolves to the same memory regardless of which tool
opened it:

- `ck` keys contexts on the **filesystem path** of the project (`~/.claude/ck/projects.json` maps
  path → context dir). Same repo at the same path = same context in both tools.
- `continuous-learning-v2` keys instincts on a **SHA-256 hash of the git remote URL** (falling back
  to local path), stored under `~/.claude/homunculus/projects/<hash>/`. Same repo = same hash =
  shared instincts.
- ECC session summaries land in `<root>/session-data/`, learned skills in `<root>/skills/learned/`,
  aliases in `<root>/session-aliases.json`. All shared once the root is shared.

So with a shared root, instincts learned in Claude Code show up when you open the repo in Cursor, and
a `/ck:save` from one tool is resumable with `/ck:resume` in the other.

**The one real risk** (the reason ECC defaults to separating them): last-write-wins. If you run both
tools on the **same project at the same time**, one can clobber the other's session summary. The fix
is simply discipline — use them **sequentially** on a given project, not concurrently. `ck` contexts
and instincts are robust to this (they're append/merge-oriented and keyed deterministically); it's
the flat session-summary files that are the collision surface. For interchangeable single-developer
use, switching tools between sessions, this is a non-issue.

### 3b. `ck` (Context Keeper) — the most reliably shareable layer

`ck`'s commands are deterministic Node scripts and its contexts are path-keyed, which makes it the
cleanest thing to share across tools. Register its SessionStart hook in **both** tools (Claude Code's
`settings.json` and Cursor's Hooks settings) so a fresh session in either picks up the same context.

Even with the hook registered, the SessionStart auto-injection has historically been unreliable, so
the safe habit is still to drive it manually:

```text
/ck:init                 # register the project (run once, from either tool)
/ck:resume <name>        # at the START of a session in either tool
/ck:save                 # before you stop, in either tool
```

Because the context is keyed on the project's filesystem path, a `/ck:save` in Claude Code is
resumable with `/ck:resume` in Cursor and vice versa — as long as the repo is at the same path and
the data root is shared (Part 3a).

**`ck` is not included in the ECC Cursor install** — copy it manually into both surfaces:

```bash
cp -r ~/ecc-source/skills/ck .cursor/skills/ck     # Cursor (required)
cp -r ~/ecc-source/skills/ck ~/.claude/skills/ck   # Claude Code (for shared memory)
```

### 3c. continuous-learning-v2 — now hook-backed on Cursor too, and naturally shared

This is the most hook-dependent skill, so it benefited most from Cursor gaining native hooks. The
`preToolUse`/`preCompact` observation events that your install registered are exactly what this skill
needs, so observation should fire on Cursor now rather than being purely emulated.

The nice property for your use case: instincts are keyed on the **git-remote hash**, not the tool. So
a repo accumulates one shared instinct set regardless of whether you were in Cursor or Claude Code
when the pattern was observed — provided the data root is shared (Part 3a). Commands work from either
tool:

```text
/instinct-status    # confirm it's actually capturing — run after a few sessions
/evolve             # cluster instincts into skills
/promote            # project-scoped → global
```

Still verify rather than assume: **run `/instinct-status` after a couple of sessions in each tool.**
The historical failure mode was hooks firing successfully but capturing nothing. Hooks being
registered (visible in Cursor's Hooks panel) is necessary but not sufficient — the proof is instincts
actually showing up.

---

## Part 4 — Understanding a codebase (ports cleanly)

Both work on Cursor since they're skill/command-driven, not hook-driven:

- **`codebase-onboarding`** — invoke conversationally ("onboard me to this codebase"). Generates
  the onboarding guide. On Cursor, have it write a project description to **`.cursor/rules/`** as an
  always-apply rule (or to `AGENTS.md`) rather than a `CLAUDE.md` — that's the file Cursor actually
  reads. This also fixes the AGENTS.md-pollution gotcha from Part 1 if you point it at your project.
- **`update-codemaps`** — generates `docs/CODEMAPS/`. Works the same; the maps are just files.

---

## Part 5 — Hooks: native on Cursor now, with caveats

Earlier this guide treated Cursor hooks as emulated and partial. That's outdated. Current Cursor has
a real **Hooks** settings panel (Settings → Hooks), and ECC registers its hooks in
`.cursor/hooks.json` as native events (`sessionStart`, `preCompact`, `afterFileEdit`, etc.). So the
quality and memory hooks from the Claude Code guide *can* run on Cursor — **if** the backend scripts
are installed and reachable.

### What now works natively (with `--profile core`)

- **`afterFileEdit`** — auto-format, typecheck, console.log warnings (via adapter → backend scripts).
- **`preCompact`** — state-saving before compaction.
- **`sessionStart`** — context load at session start.

These still respect `ECC_HOOK_PROFILE` (minimal/standard/strict) and `ECC_DISABLED_HOOKS`, same as
Claude Code, so you tune strictness the same way.

### Profile choice matters

| Profile | `hooks-runtime` module | Backend scripts | Use when |
|---|---|---|---|
| `minimal` | excluded | none copied | rules/agents/skills only; lowest context |
| `core` | included | `.cursor/scripts/hooks/` (48 files) | memory, format-on-edit, continuous-learning |

The `--profile minimal` examples elsewhere in ECC docs are for **low-context rules-only** setups. For
the hook/memory workflow this guide describes, start with **`core`**.

### The real caveats (registered ≠ effective)

1. **Hook adapter path mismatch (ECC 2.0.0, verified).** Cursor's hook adapter
   (`.cursor/hooks/adapter.js`) delegates to `<project>/scripts/hooks/*.js`. The installer copies
   backend scripts to `<project>/.cursor/scripts/hooks/` — a different path. Hooks exit 0 and
   silently no-op when the file is missing. **This is not fixed by setting `CLAUDE_PLUGIN_ROOT`** —
   the Cursor adapter resolves the project root via `path.resolve(__dirname, '..', '..')`, not that
   env var (which Claude Code hooks use).

   **Workaround after a `core` install:**

   ```bash
   mkdir -p scripts
   ln -sfn ../.cursor/scripts/hooks scripts/hooks
   test -f scripts/hooks/session-start.js && echo "hooks wired"
   ```

   Verify behavior, not just registration:

   ```bash
   echo '{}' | node scripts/hooks/session-start.js | head -5   # should print SessionStart output
   ```

2. **Registered ≠ doing something useful.** Seeing hook entries in Settings → Hooks proves they're
   wired, not that they're having an effect. Verify by behavior: does `/instinct-status` fill up?
   Does `node scripts/hooks/session-start.js` detect your project type? If yes, you're good.

### Auto-format on edit

With `preToolUse`/post-edit hooks firing, the ruff+black+mypy formatting hook can work on Cursor. But
a belt-and-suspenders approach is still worth it: also enable Cursor's own format-on-save and encode
"run ruff/black/mypy and fix issues after editing a `.py` file" as an **agent-requested rule**, so
correctness doesn't depend solely on the hook resolving its script path.

---

## Part 6 — Skills & commands (mostly port, some stubbed)

The per-feature skills work on Cursor since they're markdown workflow definitions:

| Component | Cursor status | Notes |
|---|---|---|
| `search-first` | ✓ | research-before-coding |
| `plan` | ✓ | runs inline; the `/ecc:` namespace is Claude-Code-specific, invoke by name |
| `tdd-workflow` | ✓ | RED→GREEN→refactor |
| `code-review` / `code-reviewer` agent | ✓ | agents port with full model IDs |
| `security-review` | ✓ | |
| `verification-loop` / `eval-harness` | ⚠ | the pipeline logic works; any hook-driven auto-run does not |
| `multi-*` commands | ✗ | stubbed on Cursor — multi-agent orchestration depends on Claude Code runtime |

**Invocation difference:** there's no `/ecc:` plugin namespace on Cursor. Invoke skills by name in
chat, or use `@`-mentions and manual rule references. Cursor can also load Claude skills/plugins as
**agent-decided rules** — but note those are always treated as agent-requested (Cursor decides when
they're relevant); you can't force them to always-apply or manual.

---

## Part 6b — ML/AI project-specific skills (same set, Cursor-installed)

The ML/AI lane from the Claude Code guide is unchanged in *content* — only the install location
differs. Copy the ones a given project needs into `.cursor/skills/`:

```bash
cp -r ~/ecc-source/skills/mle-workflow .cursor/skills/
cp -r ~/ecc-source/skills/cost-aware-llm-pipeline .cursor/skills/
cp -r ~/ecc-source/skills/regex-vs-llm-structured-text .cursor/skills/
cp -r ~/ecc-source/skills/llm-trading-agent-security .cursor/skills/
cp -r ~/ecc-source/skills/ai-first-engineering .cursor/skills/   # if present in your version
```

Same groupings as the Claude Code guide: core ML/LLM workflow (`mle-workflow`,
`ai-first-engineering`, `cost-aware-llm-pipeline`, `regex-vs-llm-structured-text`,
`content-hash-cache-pattern`), LLM-app security (`llm-trading-agent-security`, `security-scan`),
data/retrieval (`clickhouse-io`, `postgres-patterns`, `database-migrations`), serving/MLOps
(`api-design`, `docker-patterns`, `deployment-patterns`), the optimization pack, and deep-learning-only
(`pytorch-patterns`). Install per-project, not globally — same context-budget reasoning.

The relevant agents (`mle-reviewer`, `python-reviewer`, `database-reviewer`, `architect`,
`security-reviewer`) land in `.cursor/agents/` with full model IDs and `alwaysAllow` tool flags.

---

## Part 7 — Project instructions: `.cursor/rules/` and `AGENTS.md`, not CLAUDE.md

Cursor doesn't read `CLAUDE.md`. The equivalents:

| Purpose | Claude Code | Cursor |
|---|---|---|
| Global rules | `~/.claude/CLAUDE.md` | Cursor Settings → Rules for AI (user-level) |
| Project rules | `<repo>/.claude/CLAUDE.md` | `.cursor/rules/*.mdc` (structured) or `AGENTS.md` (simple) |

Put your model-routing + standards content into a short always-apply `.mdc`, or into a root
`AGENTS.md` if you prefer plain markdown without metadata. Keep the always-apply portion lean.

---

## Part 8 — The Cursor workflow

### One-time per project

```bash
git clone https://github.com/affaan-m/ECC.git ~/ecc-source && cd ~/ecc-source && npm install

# Discover valid component names (must run from ~/ecc-source)
npx ecc consult "mlops training model deployment" --target cursor

# Install from ~/ecc-source — use core profile for hooks/memory
./install.sh --target cursor --profile core \
  --with lang:python --with capability:machine-learning

# Wire hook backend scripts (ECC 2.0.0 path mismatch — see Part 5)
mkdir -p scripts && ln -sfn ../.cursor/scripts/hooks scripts/hooks

# ck is not included in the install — copy manually
cp -r ~/ecc-source/skills/ck .cursor/skills/ck

# Clean up: check .cursor/AGENTS.md (if present), check .cursor/agents/ for collisions
# SHARED MEMORY: do NOT set a separate Cursor root — leave both tools on the same one.
# Either rely on the default ~/.claude for both, or set the SAME explicit root in both:
# export ECC_AGENT_DATA_HOME="$HOME/.ecc-shared"

# Verify install
node ~/ecc-source/scripts/doctor.js
cat .cursor/ecc-install-state.json | head -20
echo '{}' | node scripts/hooks/session-start.js | head -3
```

Then, in Cursor chat:
```text
"onboard me to this codebase"   # codebase-onboarding -> writes project rule
/ck:init                        # register in Context Keeper
update-codemaps                 # docs/CODEMAPS/
```

### Every session

```text
/ck:resume <project-name>       # safe habit even with sessionStart hook registered
# ... work ...
/ck:save                        # before you stop — resumable from Claude Code too
git commit                      # save code (ECC never saves code)
```

### Per feature

```text
search-first <what you're integrating>
plan <feature>                  # inline planning
tdd-workflow                    # implement
code-review + security-review   # before shipping
```

---

## Part 9 — Claude Code vs Cursor: what actually changes

| Capability | Claude Code | Cursor |
|---|---|---|
| Install | plugin marketplace + manual rules | file-based installer only (`--target cursor`) |
| Config scope | mostly user-level `~/.claude/` | mostly project-level `.cursor/` |
| Rules | hierarchical markdown | `.mdc` with frontmatter + 4 activation modes |
| Hooks | 8 native event types, automatic | native Hooks panel; ECC registers 16+ entries in `.cursor/hooks.json`; needs `core` profile + path symlink |
| Auto format-on-edit | PostToolUse hook | hook fires; back it up with format-on-save + agent rule |
| Memory auto-load | SessionStart hook (already flaky) | sessionStart hook fires; still safest to `/ck:resume` manually |
| continuous-learning-v2 | hook observer + daemon | hook-backed now; verify with `/instinct-status` |
| Shared memory across both | — | yes, via shared `ECC_AGENT_DATA_HOME` + path/git-keyed stores (Part 3a) |
| Slash commands | `/ecc:*` namespace | invoke by name; `multi-*` stubbed |
| Project instructions | `CLAUDE.md` | `.cursor/rules/*.mdc` or `AGENTS.md` |

---

## Appendix — Reusable Cursor core

- **Install:** run from `~/ecc-source` — discover names with `npx ecc consult ... --target cursor`,
  then `./install.sh --target cursor --profile core --with lang:python --with capability:machine-learning`
  (or the broad `./install.sh --target cursor python`). Do **not** rely on bare `ecc install` from an
  arbitrary project cwd on a fresh machine. Never pass guessed `--modules` names.
- **Hook wiring:** after `core` install, `ln -sfn ../.cursor/scripts/hooks scripts/hooks` (Part 5).
- **Rules:** one lean always-apply `.mdc` (stack + standards), Python patterns as auto-attached globs
- **Memory (shared across both tools):** keep Cursor and Claude Code on the **same** `ECC_AGENT_DATA_HOME`
  (don't set the Cursor-specific override). Copy `ck` manually — it is not part of the ECC install.
  `ck` is path-keyed and instincts are git-remote-keyed, so the same repo resolves to the same memory
  in either tool. Drive `ck` manually (`/ck:resume` / `/ck:save`) and use the tools sequentially per
  project, not concurrently.
- **Context:** `codebase-onboarding` (writes project rule), `update-codemaps`
- **Per-feature:** search-first → plan → tdd-workflow → code-review → security-review
- **Hooks:** native on Cursor now — use `--profile core`, wire the `scripts/hooks` symlink, then verify
  with `node scripts/hooks/session-start.js`; back up format-on-edit with Cursor format-on-save plus
  an agent-requested "verify your work" rule
- **ML/LLM (project-specific):** copy into `.cursor/skills/` only on projects that need them

**Honest caveats specific to Cursor:**
1. Hooks are registered in `.cursor/hooks.json`, but *registered ≠ effective*: the adapter looks for
   `<project>/scripts/hooks/` while the installer puts scripts in `.cursor/scripts/hooks/` — symlink
   required (ECC 2.0.0). Verify by behavior (`/instinct-status`, `node scripts/hooks/session-start.js`)
   rather than assuming.
2. `npx ecc` / `ecc install` from a project directory may hit a stale global CLI — always run from
   `~/ecc-source` or use `npm exec --prefix ~/ecc-source ecc --`.
3. Shared memory works because of how the stores are keyed, but the one collision surface is the flat
   session-summary file — use the two tools sequentially on a given project, not at the same time.
4. Check `.cursor/AGENTS.md` (if present) and `.cursor/agents/` after install for collision/pollution.
5. ECC is Claude-Code-first and ships weekly; the Cursor translation still lags in places. Verify
   behavior against the actual installed files and Cursor's Hooks panel, and check ECC's issue tracker
   for open Cursor-target bugs before trusting a feature.
