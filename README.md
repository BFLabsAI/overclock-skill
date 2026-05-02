# Overclock

**Multi-agent orchestration skill for Claude Code.** Overclock turns a single Claude session into a visible workspace grid where you can spawn panes, delegate work, run models in parallel, and coordinate specialist squads — all without leaving your coding environment.

---

## What is Overclock?

Overclock is an IDE layer on top of Claude Code that exposes **7 MCP tools** for orchestrating multiple AI agents simultaneously. Each Claude session is a visible pane in a workspace grid. You can:

- **Spawn new panes** and send them independent tasks
- **Run multiple models in parallel** (Claude, Gemini, Codex, MIMO)
- **Coordinate named specialist squads** (brand, copy, cybersecurity, design, and more)
- **Track multi-milestone projects** with a visible, real-time task list

The key insight: if a task is parallelizable, blocks the current pane for more than 5 minutes, or benefits from different cognitive models on different subtasks — Overclock makes that orchestration explicit, visible, and controllable.

---

## Repository Structure

```
overclock/
├── SKILL.md                   # Master skill file — decision tree and core patterns
└── references/
    ├── tools.md               # Full API reference for all 7 Overclock tools
    ├── recipes.md             # Worked end-to-end orchestration examples
    ├── providers.md           # Provider/model selection guide with tradeoff matrix
    └── squads.md              # 8 specialist agent squads and activation patterns
```

`SKILL.md` is the entry point for Claude Code. The reference files are loaded on demand.

---

## The 7 Tools

All tools are prefixed `mcp__overclock__` in the tool registry.

| Tool | Purpose | When to call |
|------|---------|--------------|
| `pane_list` | Discover existing panes and their status | Before spawning — reuse idle panes when possible |
| `pane_list_providers` | List configured LLM providers and their models | Before spawning with any non-default provider |
| `pane_spawn` | Open a new pane in the workspace grid | When you have a parallelizable subtask |
| `pane_write` | Send a prompt to a pane as if you typed it | After spawning (or to write to an existing pane) |
| `pane_wait_idle` | Block until the target pane finishes | Always before reading — never read a running pane |
| `pane_read` | Get the last N lines of a pane's output | After `pane_wait_idle` confirms the pane is done |
| `todo_manager` | Create and update a visible, real-time task list | Projects with 3 or more distinct milestone-level tasks |

Full signatures, return shapes, and edge cases are documented in [`references/tools.md`](references/tools.md).

---

## Core Decision: When to Orchestrate

Most requests don't need orchestration. Default to handling things inline. Reach for Overclock tools only when the task meets one of these criteria:

```
Is the work parallelizable?           → spawn panes
Does it need a different model?        → spawn pane with that model
Does it have 3+ distinct milestones?   → todo_manager
Will it block this pane for >5 min?    → spawn pane, return to user immediately
Does the user say "in parallel"?       → spawn panes, no further deliberation
None of the above?                     → just do the task inline
```

Spawning a pane costs ~5–10s of setup time, and the sub-pane has zero memory of the current conversation. If a task takes 2 minutes inline, delegating is slower.

---

## The Three Core Patterns

### Pattern A: Delegate to the Worker Pane

Every orchestrator session has a pre-assigned worker pane. Its ID appears in the system prompt under `squadOrchestratorWorkerId`. Use it for execution work that would block the orchestrator.

```
pane_write(paneId: <worker-id>, text: "<self-contained task brief>")
pane_wait_idle(paneId: <worker-id>, timeoutMs: 300000)
pane_read(paneId: <worker-id>, lastN: 200)
```

The worker has no context from your session. Brief it like a colleague who just walked in: goal, input files, expected output, save location, constraints.

**Check for the worker before spawning a new pane.** Call `pane_list()` first — the orchestrator pane object has a `squadOrchestratorWorkerId` field pointing directly to the right pane.

---

### Pattern B: Parallel Fan-Out

When work splits into N independent subtasks, spawn N panes and write to all of them in **one message turn**. Writing sequentially (one pane per turn) eliminates the parallelism.

```
# Turn 1 — spawn and write in the same turn (concurrent execution starts immediately)
a = pane_spawn(model: "claude-sonnet-4-6")
b = pane_spawn(model: "claude-sonnet-4-6")
pane_write(a, "<task A brief>")
pane_write(b, "<task B brief>")

# Turn 2 — wait for both
pane_wait_idle(a, timeoutMs: 300000)
pane_wait_idle(b, timeoutMs: 300000)

# Turn 3 — read both
pane_read(a, lastN: 200)
pane_read(b, lastN: 200)
```

For very large outputs, have sub-panes save to files and read those with the standard `Read` tool — the pane buffer is bounded at ~1000 lines.

---

### Pattern C: Multi-Model Squad

Different models for different cognitive jobs in the same task. For example: Opus for architectural critique (slow, deep reasoning), Sonnet for implementation (fast, execution-oriented), Gemini for fresh web data.

**Always call `pane_list_providers()` first.** Provider IDs are machine-local — they vary between installs. Passing an unknown ID causes an error.

```
providers = pane_list_providers()
# verify that gemini-cli, codex-cli, etc. are present and note their exact IDs

opus    = pane_spawn(model: "claude-opus-4-7")
gemini  = pane_spawn(providerId: "gemini-cli", model: "gemini-2.5-flash")
mimo    = pane_spawn(providerId: "mimo-FxzXvc", model: "mimo-v2.5-pro")
```

---

## Providers and Model Selection

This install ships four providers. IDs below are machine-specific — always verify with `pane_list_providers()` at runtime.

| Provider ID | Label | Models |
|-------------|-------|--------|
| `claude-oauth` | Claude Code (default) | `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5` |
| `gemini-cli` | Gemini CLI | `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-3-flash-preview` |
| `codex-cli` | Codex CLI | `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.2` |
| `mimo-FxzXvc` | MIMO Token Plan SGP | `mimo-v2.5-pro`, `mimo-v2.5`, `mimo-v2-pro`, `mimo-v2-omni` |

### When to reach for which model

| Task | Recommended model |
|------|-------------------|
| Architecture decisions, security review, hard reasoning | **Claude Opus 4.7** |
| Feature implementation, multi-file refactors, tests | **Claude Sonnet 4.6** |
| Doc writing, mechanical transformations, log analysis | **Claude Haiku 4.5** |
| Web search, fresh facts, stylistic second opinion | **Gemini 2.5 Flash** |
| Algorithmic code, third perspective in brainstorms | **Codex GPT-5.4** |
| Cost-bounded bulk work, many parallel low-stakes panes | **MIMO Pro** |
| Multi-perspective brainstorm | Opus + Gemini + Codex together |

**Pitfalls:**
- Don't use Opus for everything "just to be safe" — it's slow and expensive. Sonnet handles most execution work just as well.
- Don't use Haiku for ambiguous tasks — it loses nuance on complex reasoning.
- Don't treat MIMO as a Claude drop-in for critical work — it's anthropic-compatible but not identical.

Full selection guide in [`references/providers.md`](references/providers.md).

---

## todo_manager: Milestone Tracking

Use `todo_manager` when a project has **3 or more distinct, milestone-level tasks**. Skip it for single-page builds, bug fixes, or conversational questions — it's overhead for trivial requests.

```
# Set up the visible task list (max 7 tasks; first becomes active immediately)
todo_manager(action: "set_tasks", tasks: [
  "Update DB schema",
  "Refactor API routes",
  "Update webhook handlers",
  "Update frontend",
  "Run end-to-end tests"
])

# Advance the list as each milestone completes — call immediately, don't batch
todo_manager(action: "move_to_task", moveToTask: "Refactor API routes")

# Signal project completion
todo_manager(action: "mark_all_done")
```

Live progress is the feature. If you batch `move_to_task` calls at the end, the user sees nothing update until the very last moment.

**Use milestone-level granularity.** "Wire up signup form" reads as progress. "Add import statement" reads as noise.

---

## Agent Squads

This install ships 8 specialist squads totaling ~100 pre-configured agents. Each squad has a chief that triages and routes to the right specialist.

| Squad | Chief | Agents | Domain |
|-------|-------|--------|--------|
| advisory-board | `@board-chair` | 11 | Strategic thinking, executive decisions |
| brand-squad | `@brand-chief` | 15 | Brand strategy and identity |
| claude-code-mastery | `@claude-mastery-chief` | 8 | Claude Code: hooks, skills, MCP, agent teams |
| copy-squad | `@copy-chief` | 23 | Copywriting — 22 legendary copywriters |
| cybersecurity | `@cyber-chief` | 15 | Offensive and defensive security operations |
| data-squad | `@data-chief` | 7 | Analytics, CLV, growth, audience |
| design-squad | `@design-chief` | 8 | Design ops and UX |
| hormozi-squad | `@hormozi-chief` | 16 | Alex Hormozi business scaling frameworks |

### Activation pattern

```
@<squad>-chief              # activate the chief — starts triage
*diagnose                   # ask the chief to triage your problem
*<workflow-name>            # run a named squad workflow
@<squad>-chief:<agent-name> # talk directly to a specific agent
```

Examples:
```
@brand-chief                          # activate brand squad
*brand-creation                       # run full brand creation workflow
@copy-chief:direct-response-writer    # talk to the direct-response specialist
```

### Squad vs. solo pane

| Situation | Use |
|-----------|-----|
| Domain matches one of the 8 squads | Activate the squad chief |
| Technical execution (build feature, fix bug) | Spawn a regular pane with `pane_spawn` |
| Cross-domain problem | Solo pane or compose multiple chiefs in parallel |

Squads are knowledge structures — they shape how Claude responds with domain expertise, frameworks, and workflows. Spawned panes are execution capacity — they do work in parallel. The two are orthogonal: you can spawn a pane and activate a squad inside it.

### Combining squads with panes

```
brand_pane = pane_spawn(model: "claude-opus-4-7")
copy_pane  = pane_spawn(model: "claude-sonnet-4-6")

pane_write(brand_pane, "@brand-chief\n*diagnose\nWe're launching SwipeScale to B2B sales...")
pane_write(copy_pane,  "@copy-chief\n*diagnose\nNeed landing page copy for SwipeScale...")
```

Each pane runs its squad independently. The orchestrator collects outputs and synthesizes.

Full squad reference in [`references/squads.md`](references/squads.md).

---

## Recipes

Four worked end-to-end examples are documented in [`references/recipes.md`](references/recipes.md):

| Recipe | When to use |
|--------|-------------|
| **Two-pane code review squad** | Critique (Opus) + remediation (Sonnet) in parallel |
| **Doc + implementation in parallel** | Both depend on the same spec, neither blocks the other |
| **Multi-perspective brainstorm** | Same prompt through Opus, Gemini, and Codex for genuine divergence |
| **Long migration with todo_manager** | Sequential milestones, worker pane does the heavy lifting, orchestrator narrates progress |

---

## Common Mistakes

1. **Reading before waiting** — `pane_read` on a running pane returns partial output. Always call `pane_wait_idle` first.
2. **Sequential fan-out** — multiple `pane_write` calls must happen in one turn, not one per turn. Sequential writes eliminate the parallelism.
3. **Vague briefs to sub-panes** — sub-panes have zero context from the current conversation. Spell out the goal, input files, expected output format, and save location.
4. **Spawning when a worker exists** — call `pane_list()` first. Reuse the idle worker pane before spawning a new one.
5. **Hardcoding provider IDs** — IDs like `mimo-FxzXvc` are machine-local and will be different on other installs. Always call `pane_list_providers()` at runtime.
6. **Micro-step todos** — `todo_manager` is for 3–7 milestone-level deliverables, not every individual action.
7. **Using todo_manager for trivial requests** — for a single-task job, the todo list is overhead.
8. **Batching `move_to_task` calls** — call it immediately when each milestone completes. The user watches the list update in real time.

---

## When NOT to Orchestrate

- Pure knowledge questions ("What is X?", "Explain Y") — answer inline
- Bug fixes, refactors, single-file edits — no parallelism benefit
- Anything a two-minute inline response handles cleanly — delegating would be slower

If in doubt, just do the task. Orchestration is a tool for specific shapes of work, not a default mode.

---

## Worker-Pane Caveat

You may be the orchestrator, or you may be a worker pane that another orchestrator is delegating to. If your initial prompt looks like a self-contained task with no orchestrator system framing, you're a worker — execute the task directly without spawning more panes.
