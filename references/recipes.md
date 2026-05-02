# Overclock Recipes — Worked Examples

End-to-end orchestration recipes. Each recipe shows the user request, the decision (why orchestration is the right move), and the exact sequence of tool calls.

---

## Recipe 1: Two-pane code review squad

### When this fits

> User: "Review the auth flow I just merged and propose fixes."

Code review benefits from two cognitive jobs done by different models:
1. **Critique** (Opus, slow + thorough): find issues, don't try to fix yet.
2. **Remediation** (Sonnet, fast + execution-oriented): given the critique, write the patch.

Doing both in one head muddles the critique with implementation bias. Splitting models gets cleaner output and they run in parallel.

### Sequence

**Turn 1 — spawn both, write to both:**

```
critic = pane_spawn(model: "claude-opus-4-7")
fixer  = pane_spawn(model: "claude-sonnet-4-6")

pane_write(critic, """
You are reviewing the auth flow at src/auth/. Read every file in that directory.
Output a structured critique to /tmp/auth-review.md with:
- Security issues (rank by severity)
- Logic bugs
- Code smell / maintainability concerns
- What's done well
DO NOT propose fixes yet. Critique only. Save the file and stop.
""")

pane_write(fixer, """
Wait 5 minutes, then read /tmp/auth-review.md (it will exist by then — another agent
is producing it). Based on each issue, propose a concrete code patch.
Output unified diffs to /tmp/auth-fixes.diff. Don't apply changes — just write the diff.
""")
```

**Turn 2 — wait for both:**

```
pane_wait_idle(critic, timeoutMs: 600000)   # 10 min — Opus is thorough
pane_wait_idle(fixer,  timeoutMs: 600000)
```

**Turn 3 — collect:**

```
critique = Read("/tmp/auth-review.md")
patches  = Read("/tmp/auth-fixes.diff")
```

Now present both to the user. The critic and fixer never share context, but the fixer reads the critique file — coordination via filesystem.

### Why this is better than inline

You'd otherwise read the auth flow, critique it, then propose fixes — all sequentially, all in one head. Two panes runs concurrently and produces cleaner separation of concerns.

---

## Recipe 2: Doc + implementation in parallel

### When this fits

> User: "Add a webhook endpoint for Stripe events. Include docs."

Docs and impl are independent — both depend on the same spec, neither blocks the other. Doing them in series wastes wall-clock time.

### Sequence

**Turn 1 — define the spec inline (this is the shared anchor), then fan out:**

First, write the spec yourself — it's small and high-leverage:

```
Write("/tmp/stripe-webhook-spec.md", """
## Stripe Webhook Endpoint

POST /api/webhooks/stripe

Events handled:
- checkout.session.completed → mark order paid
- payment_intent.payment_failed → log + notify
- customer.subscription.updated → update local sub state

Verify signature with STRIPE_WEBHOOK_SECRET via stripe.webhooks.constructEvent.
Return 200 within 5s or Stripe retries. Log all events to webhooks_log table.
""")
```

Now fan out:

```
impl_pane = pane_spawn(model: "claude-sonnet-4-6")
docs_pane = pane_spawn(model: "claude-haiku-4-5")    # docs are simpler — cheaper model fine

pane_write(impl_pane, """
Read /tmp/stripe-webhook-spec.md — that's your spec.
Implement the endpoint at src/app/api/webhooks/stripe/route.ts.
Follow the existing route patterns in src/app/api/. Add unit tests.
Run `npm run typecheck` when done. Save list of files changed to /tmp/impl-files.txt.
""")

pane_write(docs_pane, """
Read /tmp/stripe-webhook-spec.md — that's your spec.
Write user-facing docs at docs/integrations/stripe-webhooks.md covering:
- What events we handle
- How to test locally with Stripe CLI
- Production deployment checklist
- Troubleshooting common errors
Save and stop.
""")
```

**Turn 2 — wait:**

```
pane_wait_idle(impl_pane, timeoutMs: 600000)
pane_wait_idle(docs_pane, timeoutMs: 300000)
```

**Turn 3 — verify and report:**

```
files = Read("/tmp/impl-files.txt")
docs  = Read("docs/integrations/stripe-webhooks.md")
# Spot-check the implementation files in the orchestrator pane
```

### Pitfall to avoid

Don't write the spec inside one of the sub-panes and have the other "wait for it". Filesystem coordination across panes works but adds fragility (timing, partial reads). When something is shared input for multiple panes, write it from the orchestrator first, then fan out.

---

## Recipe 3: Multi-perspective brainstorm

### When this fits

> User: "I'm naming a new product. Help me brainstorm — give me a few angles."

Brainstorms get repetitive when one model does them. Different models have different priors, training, and rhetorical defaults. Run the same prompt through 3 models in parallel; the diversity is the value.

### Sequence

**Turn 1 — list providers (machine-specific IDs), then spawn 3 panes:**

```
providers = pane_list_providers()
# Confirm: claude-oauth, gemini-cli, codex-cli all present

opus    = pane_spawn(model: "claude-opus-4-7")
gemini  = pane_spawn(providerId: "gemini-cli", model: "gemini-2.5-flash")
codex   = pane_spawn(providerId: "codex-cli",  model: "gpt-5.4")

prompt = """
I'm naming a new product. It's a SwipeScale-style lead-rating tool —
swipe left/right on inbound leads, AI learns your preferences over time.
Target audience: B2B sales teams.

Generate 10 product names. For each, give:
- The name
- The angle (what it evokes)
- Why it might fail

Be willing to be weird. Save to /tmp/names-<your-model>.md.
"""

pane_write(opus,   prompt + "\nUse filename /tmp/names-opus.md")
pane_write(gemini, prompt + "\nUse filename /tmp/names-gemini.md")
pane_write(codex,  prompt + "\nUse filename /tmp/names-codex.md")
```

**Turn 2 — wait for all:**

```
pane_wait_idle(opus,   timeoutMs: 300000)
pane_wait_idle(gemini, timeoutMs: 300000)
pane_wait_idle(codex,  timeoutMs: 300000)
```

**Turn 3 — read all three, synthesize:**

```
opus_names    = Read("/tmp/names-opus.md")
gemini_names  = Read("/tmp/names-gemini.md")
codex_names   = Read("/tmp/names-codex.md")
```

Now your job in the orchestrator pane is the synthesis: "Here are 30 candidates from 3 models. The strongest 5, with reasoning across perspectives:"

### Why this beats one big prompt

A single model self-correlates — its 30 names will share rhetorical DNA. Three models give you genuine divergence. The synthesis step (which only the orchestrator can do, because only it sees all three outputs) is where the value lands.

---

## Recipe 4: Long-running migration with todo_manager

### When this fits

> User: "Migrate the payment system from Stripe Checkout to Stripe Payment Intents. We need to update the API, the DB schema, the webhooks, and the frontend."

This is a 4-milestone project. The user wants visible progress. Sequential, each step depends on the prior, but a worker pane can do the heavy lifting while the orchestrator narrates.

### Sequence

**Turn 1 — set up the visible task list:**

```
todo_manager(action: "set_tasks", tasks: [
  "Update DB schema for Payment Intents",
  "Refactor API routes to use PaymentIntents",
  "Update webhook handlers",
  "Update frontend checkout flow",
  "Run end-to-end tests"
])
```

**Turn 2 — delegate task 1 to the worker:**

```
worker = "T06AfShP"   # from system prompt; or use pane_list() to discover

pane_write(worker, """
Migration task 1 of 5: DB schema update for Stripe Payment Intents.

Current: orders table has stripe_session_id, stripe_checkout_url
Target:  add stripe_payment_intent_id, stripe_client_secret; deprecate the old fields
         (don't drop yet — we'll do that after rollout)

Generate the Supabase migration in supabase/migrations/. Don't apply it.
Save the migration filename to /tmp/migration-1-file.txt.
Output a one-paragraph summary to /tmp/migration-1-summary.md.
""")
pane_wait_idle(worker, timeoutMs: 600000)
result1 = pane_read(worker, lastN: 200)
```

**Turn 3 — verify, advance the todo, delegate task 2:**

```
summary1 = Read("/tmp/migration-1-summary.md")
# Quick orchestrator-side check that the migration looks right

todo_manager(action: "move_to_task", moveToTask: "Refactor API routes to use PaymentIntents")

pane_write(worker, """
Migration task 2 of 5: Refactor API routes.

Read the migration created in step 1: $(cat /tmp/migration-1-file.txt)
Update src/app/api/checkout/* to create PaymentIntents instead of Checkout sessions.
Preserve the existing public API surface where possible — minimize breaking changes.

Files modified → /tmp/migration-2-files.txt
Summary → /tmp/migration-2-summary.md
""")
pane_wait_idle(worker, timeoutMs: 900000)   # 15 min — API refactor is bigger
```

**Continue the pattern through task 5.** Between each, the orchestrator:
1. Reads the worker's summary
2. Surfaces anything notable to the user (or asks for input if there's a decision)
3. Calls `move_to_task` so the visible list advances
4. Delegates the next task

**Final turn:**

```
todo_manager(action: "mark_all_done")
```

### Why use the worker pane instead of doing it inline

Each task is 5-15 minutes of focused work — that's a long time for the orchestrator to be silent. The worker does the work; the orchestrator stays free to surface progress, answer user questions, and make decisions between tasks. The user sees the todo list advance in real time and gets summaries from the orchestrator without waiting on a 60-minute monologue.

---

## Choosing between recipes

| Situation | Recipe |
|-----------|--------|
| 2+ independent perspectives on same input | Recipe 3 (multi-model brainstorm) |
| Different cognitive jobs on related inputs | Recipe 1 (critique + fix) |
| Independent work products, shared spec | Recipe 2 (doc + impl in parallel) |
| Sequential project, visible milestones, heavy work | Recipe 4 (worker + todo_manager) |
| One small task | None — just do it inline |

The shape of the work picks the recipe. If none of these patterns fit, you probably don't need orchestration.
