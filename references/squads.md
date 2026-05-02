# Agent Squads

Squads are pre-defined teams of specialist agents, each with a designated chief that triages and routes. They live in `agents-squads/` in the project root. This install ships 8 squads totaling ~100 specialist agents.

---

## Installed squads

| Squad | Chief | Agents | Domain |
|-------|-------|--------|--------|
| advisory-board | `@board-chair` | 11 | Strategic thinking, executive decisions, board-level guidance |
| brand-squad | `@brand-chief` | 15 | Brand strategy — 10 legendary brand thinkers + 4 specialists + chief |
| claude-code-mastery | `@claude-mastery-chief` | 8 | Claude Code expertise: hooks, skills, MCP, subagents, agent teams |
| copy-squad | `@copy-chief` | 23 | Copywriting — 22 legendary copywriters + chief |
| cybersecurity | `@cyber-chief` | 15 | Offensive + defensive security operations |
| data-squad | `@data-chief` | 7 | Analytics, CLV, growth, community, customer success, audience |
| design-squad | `@design-chief` | 8 | Design ops — 3 experts + 4 specialists + chief |
| hormozi-squad | `@hormozi-chief` | 16 | Alex Hormozi business scaling frameworks |

---

## Activation pattern

Each squad has a chief that does triage — start there.

```
@<squad>-chief                # activate
*diagnose                     # ask the chief to triage your problem
*<workflow-name>              # run a named workflow
@<squad>-chief:<agent-name>   # talk directly to a specific agent
```

Examples:
```
@brand-chief                            # activate brand squad chief
*brand-creation                         # full brand creation workflow
@brand-chief:brand-storyteller          # talk directly to the storyteller agent
```

The chief reads your request, picks the right specialist, and routes. You usually don't need to know specific agent names upfront.

---

## Squad anatomy

Each squad directory contains:

```
agents-squads/<squad-name>/
├── config.yaml         # Squad definition (or squad.yaml)
├── README.md           # Quick start
├── agents/             # Agent persona .md files
├── workflows/          # Multi-step workflow definitions
├── tasks/              # Specific task definitions
├── checklists/         # Quality / process checklists
└── data/               # Reference data the squad uses
```

`config.yaml` defines:
- **Tier structure** — Tier 0 (chief), Tier 1 (core specialists), Tier 2 (strategic/cross-cutting)
- **Handoff matrix** — `routes_to`, `collaborates_with`, `escalates_to` per agent
- **Cross-cutting concerns** — shared context files, knowledge updates, quality standards
- **Activation prefix** — usually the squad name; `@brand-chief` etc.

---

## Squad vs solo Overclock pane: which to use?

| Situation | Use |
|-----------|-----|
| Domain is one of the 8 squads above | Activate the squad chief — they triage faster than you can route manually |
| Domain is technical execution (build feature, fix bug) | Spawn a regular pane via `pane_spawn` |
| Need a cross-domain answer | Solo pane (no squad covers it) or compose multiple chiefs in parallel |
| User asks for a specific squad agent | Activate that squad |

Squads are knowledge structures — they shape how you respond. Spawned panes are execution capacity — they do work in parallel. The two are orthogonal: you can spawn a pane and then activate a squad inside it.

---

## When the user types `@some-name`

Treat any `@xxx-chief` or `@xxx-name:agent-name` as a squad activation request. The squad's `config.yaml` defines the activation behavior — read it (`agents-squads/<name>/config.yaml`) and follow the agent's `activation-instructions` block in the relevant `.md` file under `agents/`.

The pattern is consistent across squads:
1. STEP 1: read the entire agent file (it contains the full persona)
2. STEP 2: adopt the persona
3. STEP 3: display the configured greeting
4. STEP 4: HALT and await user input

Don't load multiple agents at once — the chief routes if needed.

---

## Why squads exist

Solo Claude is general-purpose. Squads encode specialist personas (e.g. "brand storyteller in the tradition of Marty Neumeier") with their own frameworks, voice, and decision criteria. For domain-specific work, the framing shifts the output meaningfully — a brand exercise run through `@brand-chief` produces strategically different recommendations than a generic prompt would.

Squads also provide curated workflows (multi-step processes) and checklists that encode best practices from the domain. Activating the chief inherits all of that.

---

## Squad-pane composition

You can combine: spawn a pane and activate a squad inside it. Useful when:
- You want a squad to run while you continue other work in the orchestrator
- You're running multiple squads in parallel (e.g., brand + copy on the same launch)

```
brand_pane = pane_spawn(model: "claude-opus-4-7")
copy_pane  = pane_spawn(model: "claude-sonnet-4-6")

pane_write(brand_pane, "@brand-chief\n*diagnose\nWe're launching SwipeScale to B2B sales teams...")
pane_write(copy_pane,  "@copy-chief\n*diagnose\nNeed launch landing page copy for...")
```

Each pane runs its squad independently; the orchestrator collects outputs and synthesizes.
