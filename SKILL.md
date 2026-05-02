---
name: overclock
description: Master skill for Overclock multi-agent orchestration IDE. Use this skill whenever you need to spawn panes, delegate work to agents, manage parallel execution, coordinate multi-model squads, use the todo manager, or do anything involving pane_spawn, pane_write, pane_read, pane_wait_idle, pane_list, pane_list_providers, or todo_manager. Trigger on any mention of: opening a new pane, running tasks in parallel, delegating to another agent, spawning Opus/Gemini/Codex/MIMO, checking pane status, reading pane output, structuring a multi-step project with visible milestones, or activating an agent squad like @brand-chief, @cyber-chief, @copy-chief, @hormozi-chief, etc.
---

# Overclock

You are running inside Overclock — a multi-agent orchestration IDE. Each Claude session is a visible pane in a workspace grid. You can spawn more panes, delegate work to them, run multiple models in parallel, and coordinate squads of named specialist agents.

This file covers the decision tree and must-know patterns. For depth, read the relevant reference file:

- **references/tools.md** — full API for the 7 Overclock tools, parameter shapes, return values
- **references/recipes.md** — worked end-to-end examples (code review squad, parallel doc+impl, multi-perspective brainstorm, long migration)
- **references/providers.md** — which provider/model to reach for, with cost & latency tradeoffs
- **references/squads.md** — the 8 specialist squads installed in this project and how to activate them

Read these only when needed. SKILL.md alone is enough for most decisions.

---

## The first decision: do anything special at all?

Most user requests don't need orchestration. Default to handling things inline. Reach for Overclock tools only when the task genuinely benefits from one of these signals:

```
Is the work parallelizable?           → spawn panes
Does it need a different model?        → spawn pane with that model
Does it have 3+ distinct milestones?   → todo_manager
Will it block this pane for >5 min?    → spawn pane, return to user immediately
Does the user say "in parallel"?       → spawn panes, no further deliberation
None of the above?                     → just do the task. Don't perform orchestration.
```

Spawning a pane has cost: setup time (~5-10s), separate context (sub-pane has zero memory of this conversation), and you have to brief it well. If a task takes you 2 min inline, delegating is slower, not faster.

---

## The 7 tools at a glance

| Tool | Purpose | When to call |
|------|---------|--------------|
| `pane_list` | Discover existing panes | Before spawning — reuse idle panes when possible |
| `pane_list_providers` | List configured LLMs | Before spawning with a non-default provider |
| `pane_spawn` | Open new pane | Got a parallelizable subtask |
| `pane_write` | Send prompt to pane | After spawn (or to existing pane) |
| `pane_wait_idle` | Block until pane done | Before reading — never read a running pane |
| `pane_read` | Get pane output | After wait_idle |
| `todo_manager` | Visible task list | Projects with 3+ milestones |

Full signatures, return shapes, and edge cases live in `references/tools.md`.

---

## The three must-know patterns

### Pattern A: delegate to the worker pane (most common)

You always have a worker pane available — its ID is at the top of your system prompt under "squad orchestrator worker". Default `T06AfShP`. Use it for execution work that would block this pane.

```
pane_write(paneId: <worker-id>, text: "<self-contained task>")
pane_wait_idle(paneId: <worker-id>, timeoutMs: 300000)   # 5 min for non-trivial
pane_read(paneId: <worker-id>, lastN: 200)
```

The worker has no memory of this conversation. Brief it like a colleague who just walked in: goal, files involved, expected output, where to save it.

### Pattern B: parallel fan-out

Work splits into N independent subtasks. Spawn N panes, write to all in **one message turn**, then collect.

```
# Turn 1 — spawn + write together (concurrent execution)
a = pane_spawn(model: "claude-sonnet-4-6")
b = pane_spawn(model: "claude-sonnet-4-6")
pane_write(a, "<task A>")
pane_write(b, "<task B>")

# Turn 2 — wait
pane_wait_idle(a)
pane_wait_idle(b)

# Turn 3 — read
pane_read(a); pane_read(b)
```

If you spawn-write-wait-read sequentially per pane, you've lost the parallelism. The whole point is concurrent execution.

### Pattern C: multi-model squad

Different models for different cognitive jobs in the same task. **Always call `pane_list_providers()` first** — provider IDs are machine-specific and unknown IDs are rejected.

```
opus    = pane_spawn(model: "claude-opus-4-7")              # architecture / hard reasoning
gemini  = pane_spawn(providerId: "gemini-cli", model: "...") # web search / fast iteration
mimo    = pane_spawn(providerId: "mimo-FxzXvc", model: "...") # cheap bulk work
```

See `references/providers.md` for which model to reach for in which situation.

---

## todo_manager: when, how, and why

Use when the project has **3+ distinct, milestone-level tasks** ("Build auth", "Add database", "Create dashboard"). Skip for single-page builds, trivial requests, or conversational questions.

```
todo_manager(action: "set_tasks", tasks: [...])    # max 7 — first becomes in_progress
todo_manager(action: "move_to_task", moveToTask: "Add database")  # call as soon as a task completes
todo_manager(action: "mark_all_done")              # at the end
```

Why milestone-level only: micro-steps create noise, the user scans for progress at a glance. "Wire up signup form" reads as progress; "Add import statement" reads as filler.

Call `move_to_task` immediately when each milestone completes — don't batch. Live progress is the feature.

---

## When NOT to invoke this skill

- "What is X?" / "explain Y" — pure knowledge questions, no orchestration involved
- Bug fixes, refactors, single-file edits — inline work, no parallelism
- Anything where the user is mid-conversation about an unrelated topic and just mentions "open" or "pane" or "agent" in passing

The skill triggers when there's actual orchestration work. If in doubt, just do the task.

---

## Common mistakes

1. **Reading before waiting** — `pane_read` on a running pane returns partial output. Always `pane_wait_idle` first.
2. **Sequential when parallel was possible** — multiple `pane_write`s belong in one turn, not one per turn.
3. **Vague briefs to sub-panes** — they have zero context. Spell out files, goals, output format, save path.
4. **Spawning when worker exists** — call `pane_list()` first, reuse the idle worker.
5. **Non-default provider without listing first** — IDs vary per machine; unknown IDs error out.
6. **Micro-step todos** — todo_manager is for 3-7 milestones, not every action.
7. **Using todo_manager for single-task work** — it's overhead for trivial requests.
8. **Forgetting `move_to_task`** — the user can't see live progress if you batch updates.

---

## Worker-pane caveat

You may be the orchestrator, but you may also be a worker pane that another orchestrator is delegating to. If your initial prompt looks like a self-contained task with no system framing about being an orchestrator, you're a worker — execute the task directly, don't try to spawn more panes for it.
