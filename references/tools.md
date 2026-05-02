# Overclock Tools — Full API Reference

The 7 MCP tools exposed by Overclock. All are prefixed `mcp__overclock__` in the tool registry.

---

## `pane_spawn`

Open a new Claude pane in the workspace grid. The pane is visible to the user.

**Signature:**
```
pane_spawn(
  providerId?: string,   // omit for default 'claude-oauth'
  model?: string,        // must be a model the provider actually exposes
  cwd?: string           // absolute path; defaults to current workspace
) → { paneId: string, ... }
```

**Returns:** the new `paneId`. Save it — every subsequent `pane_write` / `pane_read` / `pane_wait_idle` needs it.

**Critical rule:** if `providerId` is anything other than `claude-oauth`, call `pane_list_providers()` first. Provider IDs are machine-local (e.g. `mimo-FxzXvc` on this machine, but might be `mimo-Abc123` elsewhere). Passing an unknown ID returns an error.

**Common pitfalls:**
- Passing a model the provider doesn't expose → error. Verify against `pane_list_providers()` output.
- Passing `cwd` outside the workspace → may work but file paths get confusing for the spawned pane.
- Spawning when `pane_list()` already shows an idle pane you could reuse → wasted setup time.

---

## `pane_list`

List every active pane in the workspace.

**Signature:**
```
pane_list() → {
  ok: bool,
  count: number,
  panes: [{
    paneId: string,
    agent: 'claude',
    cwd: string,
    status: 'running' | ...,
    providerId?: string,
    providerType?: 'oauth' | 'anthropic-compat',
    providerLabel?: string,
    model?: string,
    squadOrchestratorWorkerId?: string  // present on orchestrator panes — value is the worker pane's ID
  }]
}
```

**When to call:** before spawning, to discover existing panes you could reuse. Especially relevant when looking for the worker pane.

**Notes:**
- The default provider (`claude-oauth`) often has no `providerId` field on the pane object — its presence indicates a non-default provider.
- `status` reflects current activity. A "running" pane is busy; an idle pane has the Claude prompt waiting for input.

---

## `pane_list_providers`

List every LLM provider configured in this Overclock install.

**Signature:**
```
pane_list_providers() → {
  ok: bool,
  count: number,
  providers: [{
    id: string,        // pass this as providerId to pane_spawn
    label: string,     // human-readable name shown in UI
    type: 'oauth' | 'anthropic-compat',
    host: string,
    models: string[]   // valid model values for this provider
  }]
}
```

**Critical rule:** call this whenever you intend to use a non-default provider. The returned `id` and `models` are authoritative — anything you pass to `pane_spawn` must match these exactly.

---

## `pane_write`

Type text into another pane's terminal as if the user typed it.

**Signature:**
```
pane_write(
  paneId: string,
  text: string,
  submit?: boolean   // default true — appends Enter. False stages the text without executing.
) → { ok: bool }
```

**Briefing principle:** the target pane has no memory of this conversation. Write self-contained briefs that include:
- Goal: what done looks like
- Inputs: file paths, data, prior context
- Output: what to produce, in what format
- Save location: where to write results so the orchestrator can read them
- Any constraints (don't touch X, must use Y)

**`submit=false`** is rare. Use it only when you want to stage a long prompt and let the user inspect/edit before running.

---

## `pane_wait_idle`

Block until the target pane returns to idle (Claude prompt re-appears).

**Signature:**
```
pane_wait_idle(
  paneId: string,
  timeoutMs?: number   // default 120000 (2 min)
) → { ok: bool, idle: bool }
```

**When to call:** always between `pane_write` and `pane_read`. Reading a running pane gives partial output.

**Timeouts:**
- Quick lookups / single-file edits: default 120s is fine.
- Multi-file generation, installs, build steps: bump to 300000 (5 min) or 600000 (10 min).
- If the timeout fires and the pane is still running, the call returns `idle: false`. You can re-call to wait longer, or `pane_read` for partial progress.

---

## `pane_read`

Read the last N lines from a pane's terminal output buffer (ANSI escape codes already stripped).

**Signature:**
```
pane_read(
  paneId: string,
  lastN?: number   // default 100. Max ~1000 (buffer cap)
) → { ok: bool, lines: string[] | string }
```

**When to use larger `lastN`:** if you asked the pane to produce a long report inline, default 100 lines may truncate. For very large outputs, the better pattern is to have the sub-pane save to a file and read the file with the standard `Read` tool — the buffer is bounded.

**ANSI handling:** already stripped — you get clean text. No need to filter escape codes.

---

## `todo_manager`

Manages a structured, user-visible task list for milestone tracking.

**Signature:**
```
todo_manager(action, ...) where action is one of:

  set_tasks(tasks: string[], taskNameActive?: string, taskNameComplete?: string)
    — max 7 tasks. First becomes in_progress immediately.

  move_to_task(moveToTask: string, taskNameActive?: string, taskNameComplete?: string)
    — completes all prior tasks, sets the named task to in_progress.

  add_task(task: string)
    — appends a single task to the existing list.

  read_list()
    — view current state without changes.

  mark_all_done()
    — completes everything; signals project end.
```

**`taskNameActive` / `taskNameComplete`:** 2-5 word UI labels for the task while running and after completion. Optional; omit for default.

**Use when:** projects have 3+ distinct, milestone-level deliverables. Examples: "Build auth", "Set up database", "Create dashboard", "Wire payment flow".

**Skip when:**
- Single cohesive build (one landing page, one form, one component)
- Trivial conversational requests
- Bug fixes, refactors, single-file edits

**Live progress matters:** call `move_to_task` immediately when each milestone is done. Don't batch — the user is watching the list update.

---

## Pane object cheat sheet

What you get back from `pane_list`:

| Field | Meaning |
|-------|---------|
| `paneId` | Unique ID — use for write/read/wait |
| `agent` | Always `"claude"` |
| `cwd` | Working directory |
| `status` | `"running"` while active |
| `providerId` | Present only when non-default provider |
| `providerType` | `"oauth"` (Claude/Gemini/Codex) or `"anthropic-compat"` (MIMO) |
| `providerLabel` | Display name (e.g. `"MIMO Token Plan SGP"`) |
| `model` | Model ID; absent if provider's default is in use |
| `squadOrchestratorWorkerId` | On orchestrator panes — value is the worker pane's `paneId` |

The presence of `squadOrchestratorWorkerId` is how you identify orchestrator panes. The value points to the worker the orchestrator should delegate to.
