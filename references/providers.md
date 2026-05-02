# Providers & Model Selection

Which provider/model to reach for in which situation. Always run `pane_list_providers()` at runtime to confirm IDs and available models — values below are a snapshot from this install and may drift.

---

## Snapshot of this install

| Provider ID | Label | Type | Models |
|-------------|-------|------|--------|
| `claude-oauth` | Claude Code | oauth | `claude-sonnet-4-6`, `claude-opus-4-7`, `claude-haiku-4-5` |
| `gemini-cli` | Gemini CLI | oauth | `gemini-3-flash-preview`, `gemini-3.1-flash-lite-preview`, `gemini-2.5-flash`, `gemini-2.5-flash-lite` |
| `codex-cli` | Codex CLI | oauth | `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, `gpt-5.2` |
| `mimo-FxzXvc` | MIMO Token Plan SGP | anthropic-compat | `mimo-v2.5-pro`, `mimo-v2.5`, `mimo-v2-pro`, `mimo-v2-omni` |

**Default provider:** `claude-oauth`. Omit `providerId` in `pane_spawn` to use it.

---

## When to reach for which model

Match the work to the model. Cost, speed, and cognitive strength differ.

### Claude Opus 4.7 — `claude-opus-4-7`
- **Best for:** architectural decisions, security-critical code review, hard reasoning, ambiguous spec interpretation
- **Tradeoff:** slowest and most expensive. Don't use for execution work where Sonnet would do the same job
- **Reach for it when:** the task hinges on getting a judgment call right

### Claude Sonnet 4.6 — `claude-sonnet-4-6`
- **Best for:** general-purpose implementation, multi-file refactors, test writing, the bulk of orchestration work
- **Tradeoff:** the workhorse — good at almost everything, exceptional at nothing
- **Reach for it when:** in doubt; this is the default

### Claude Haiku 4.5 — `claude-haiku-4-5`
- **Best for:** simple parallel sweeps, doc generation, mechanical transformations, log analysis
- **Tradeoff:** loses nuance on complex reasoning
- **Reach for it when:** the task is well-specified, repetitive, and you're spawning many panes for it

### Gemini 2.5 Flash — `gemini-2.5-flash` (provider: `gemini-cli`)
- **Best for:** web search–heavy tasks, fast iteration, fresh information
- **Tradeoff:** different style/priors than Claude — useful for a second opinion
- **Reach for it when:** task involves "look up current X" or you want stylistic divergence

### Codex (GPT) — `gpt-5.4` (provider: `codex-cli`)
- **Best for:** algorithmic code, classical CS problems, third-perspective in brainstorms
- **Tradeoff:** different ergonomics; output style differs from Claude/Gemini
- **Reach for it when:** classic algo/data-structure work, or you want a third voice in an A/B/C comparison

### MIMO — `mimo-v2.5-pro` (provider: `mimo-FxzXvc`)
- **Best for:** cost-efficient bulk work where token budget matters
- **Tradeoff:** anthropic-compat shim; behaves close to Claude but not identical
- **Reach for it when:** spawning many panes for parallel low-stakes work, or running long sweeps where the Claude bill would be painful

---

## Decision matrix

| Task | Recommended |
|------|-------------|
| Architecture / hard design call | Opus 4.7 |
| New feature implementation | Sonnet 4.6 |
| Single-file edit, refactor | Sonnet 4.6 (or current pane inline) |
| Test generation | Sonnet 4.6 |
| Doc writing | Haiku 4.5 |
| 10+ parallel mechanical edits | Haiku 4.5 or MIMO |
| Web search / fresh facts | Gemini 2.5 Flash |
| Multi-perspective brainstorm | Opus + Gemini + Codex (one each) |
| Security review | Opus 4.7 |
| Fix-it after critique | Sonnet 4.6 |
| Long-running migration step | Sonnet 4.6 in worker pane |
| Cost-bounded bulk transform | MIMO Pro |

---

## Calling pattern by provider type

**Default (Claude):**
```
pane_spawn(model: "claude-sonnet-4-6")
```

**Non-default (anything else):**
```
providers = pane_list_providers()
# verify the id and model exist
pane_spawn(providerId: "<id>", model: "<model from that provider's list>")
```

Skipping `pane_list_providers()` is the most common error — IDs are machine-specific.

---

## Pitfalls

1. **Hardcoding provider IDs from another machine** — `mimo-FxzXvc` here might be `mimo-Xy7zKp` elsewhere. Always list at runtime.
2. **Using Opus for everything "just to be safe"** — slow and expensive. Sonnet handles most work; reach for Opus only when reasoning depth matters.
3. **Using Haiku for ambiguous tasks** — it loses nuance. Better for well-specified mechanical work.
4. **Treating MIMO as a Claude drop-in** — it's anthropic-compatible but not identical. Verify outputs on critical work.
5. **Mixing models randomly** — diversity helps brainstorms; for execution, pick one model per coherent task.
