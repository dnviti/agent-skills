---
name: complex-work
description: >-
  Orchestrate a complex, multi-phase engineering task from planning through
  delivery using your coding agent's memory/rules hierarchy and (where available)
  subagents, with cost-tiered model routing that defaults to the cheapest capable
  tier and escalates only on evidence. Provider- and tool-agnostic: works with any
  LLM (Claude, GPT, Gemini, Llama, or other open-weight models) and any agentic
  coding system (Claude Code, Cursor, Aider, Cline, Codex, Windsurf, and others).
  Use when the user invokes /complex-work, or asks to plan-and-execute a substantial
  task, break work into phases with explicit completion tests, run an
  orchestrator/subagent workflow, or wants adversarial review and re-planning built
  in. Not for quick one-off edits, single-file changes, or simple questions.
---

# /complex-work — Orchestrated Multi-Phase Execution

You are the **orchestrator** for the task below. You rely on your coding agent's
memory/rules system — whatever your host tool provides — for context and persistence.

This skill is deliberately **provider- and tool-agnostic**: it names model *tiers* by
role, not by vendor, and describes host features (memory, subagents, plan mode, guardrails)
by capability so you can map them onto whatever your environment actually offers. Wherever
a concrete mechanism is named, treat it as an example of a category — use your tool's
equivalent.

Operating principles:

- **Work in phases.** Each phase has a purpose, inputs, an output artifact, and an
  explicit **completion test** that proves it's done.
- **Keep the main context lean.** Delegate verbose or noisy work (research, big greps,
  test runs, log/doc scraping, adversarial review) to subagents *if your tool supports
  them*; otherwise run that work yourself and keep only the distilled summary in your
  working context.
- **Match model tier to *judgment difficulty*, not importance.** Default every phase to
  the cheapest tier that could plausibly pass its completion test; escalate only when a
  test actually fails or the work is demonstrably beyond the current tier. A phase can be
  critical to the project but mechanical to run — that is still a floor-tier job.
- **Memory is context, not enforcement.** If a rule *must* hold, back it with a mechanical
  guardrail (a hook, a deny-rule, a pre-commit/CI check, a sandbox), not a memory line
  (see §4).

> This skill names model tiers by role, not by vendor. Model lineups, aliases, and pricing
> change fast and differ across providers — confirm the exact models available in your
> environment and their current prices before pinning them to phases.

---

## 0. Task

Use whatever the invoker provided (inline in their message, or in the block below). If
invoked as a command/skill with arguments, treat those arguments as the task. If **GOAL**,
**CONTEXT & TOOLS**, or **DONE WHEN** is missing and you can't infer it, ask up to **3**
questions, then proceed — and note the gap so it can be added to memory later.

```
=== TASK ===
GOAL:            [what to build/do, the final output, and why it matters]
CONTEXT & TOOLS: [repos, docs, credentials, environments, constraints — or "see project memory"]
DONE WHEN:       [the concrete condition(s) that mean this is finished]
=== END TASK ===
```

---

## 1. Set your orchestrator (main) model — do this at launch

Your resident session model is usually a launch-time choice — most agentic tools don't let
you cleanly self-switch your own main model mid-run, so pick deliberately up front.

- **Default to a top-tier model** for genuinely large, ambiguous, or multi-phase tasks.
  Long-running, investigate-before-acting, verify-your-own-work orchestration is where the
  most capable reasoning models earn their cost.
- **Drop to a strong flagship (one notch below the absolute top)** when the task is
  well-scoped with an unambiguous DONE-WHEN and the plan is mostly dispatch — often the
  same capability class for well-scoped agentic work at a fraction of the cost, and the
  orchestrator session stays resident across every phase.

Set the model however your tool allows — a CLI flag at launch (e.g. `--model <id>`), a
config file, or an in-session command. In your opening restatement, **state your resident
model and (if exposed) reasoning-effort level**, and pin explicit model identifiers — not
vague aliases — for every phase and subagent.

---

## 2. Load context from memory/rules (before anything else)

Treat your agent's memory/rules hierarchy as your source of truth. The exact files depend
on your tool — common conventions include:

- **Project rules** — repo conventions, build/test commands, architecture. E.g.
  `CLAUDE.md` / `.claude/rules/` (Claude Code), `AGENTS.md` (the emerging cross-tool
  convention), `.cursor/rules/` or `.cursorrules` (Cursor), a conventions file such as
  `CONVENTIONS.md` (Aider), `.clinerules` (Cline), `.windsurfrules` (Windsurf), or a repo
  `README` / `docs/`.
- **User / global defaults** — your cross-project preferences (a home-directory rules
  file, global config).
- **Accumulated learnings** — any auto/persistent memory your tool keeps across sessions.

On conflict, **more specific wins**: managed/org policy > project > user; accumulated
learnings are lowest.

Then:

1. **Reconcile the TASK against memory.** Restate in 3–5 lines: the goal, the definition of
   done, hard constraints, and which key conventions/commands you're pulling from project
   vs user memory.
2. **Map the code with a read-only exploration pass.** Delegate this to a read-only
   subagent at the floor tier if your tool supports subagents; otherwise do a lean,
   targeted scan yourself and keep only the summary. Do **not** record anything derivable
   from `grep`/`git` as memory.

---

## 3. Plan (read-only — no edits yet)

- Break the work into **phases**. For each: purpose, inputs, output artifact, and an
  explicit **completion test** (a command/check that proves it's done).
- Mark **independent (parallelizable)** vs **dependent** phases and state the order.
- Assign an **executor** per phase: the main agent, or a subagent (if available) for
  token-heavy / noisy work and for adversarial review. Subagents may keep their own memory.

If your tool has a dedicated **plan / read-only mode**, use it here; otherwise simply make
no edits until the plan is set.

### Model routing (default down, escalate on evidence)

Start each phase at the cheapest tier that could pass its completion test and escalate
**Floor → Mid → High → Top** only when a test fails or the work is demonstrably beyond the
current tier. Never pre-assign a high tier "to be safe." Reserve expensive tokens for
genuinely hard reasoning plus adversarial review of critical logic.

The four tiers, by role (map them to your provider's lineup — see below):

- **Floor tier — the cheapest, fastest small model.** High-volume, low-judgment, ideally
  read-only work: codebase mapping, big greps, scanning logs and verbose tool output,
  summarizing noisy results, mechanical text transforms, formatting, boilerplate
  scaffolding. **Run the read-only exploration pass here** — its whole purpose is keeping
  the resident session's context clean. Cheapest tokens; the throughput workhorse.
- **Mid tier — the standard coding model.** Standard implementation, most refactors, test
  writing, straightforward debugging. The large majority of real coding should land here.
- **High tier — a strong reasoning model.** Tricky debugging, security-sensitive logic,
  non-trivial or cross-cutting implementation, and standard adversarial review.
- **Top tier — the most capable (and most expensive) model.** Orchestration and the
  genuinely hardest work only: hard architecture, ambiguous or high-stakes design, and
  adversarial review of critical logic.

**Relative token cost, cheapest first:** Floor ≪ Mid < High (~2× Mid) < Top (~2× High).
Exact ratios differ by provider, but the shape holds. Most work should sit at the bottom
two tiers.

**Map tiers to your provider.** Every major lineup has this small → mid → large → frontier
shape; slot your available models into the four roles. For example, Anthropic's Claude line
maps as Haiku (floor) → Sonnet (mid) → Opus (high) → its top reasoning model; OpenAI's GPT
line, Google's Gemini line (Flash-Lite/Flash/Pro tiers), and open-weight families (e.g.
Llama, Qwen, DeepSeek at increasing sizes) all expose the same gradient. Use whatever tiers
you actually have — if your environment only offers two models, collapse the ladder onto
those two and note it.

**Second lever — effort.** If your model exposes a reasoning-effort / thinking-budget
control, keep it at default (or lower) on cheap mechanical phases and reserve high effort
for hard-reasoning phases and the adversarial review. Effort burns tokens too.

**Routing hygiene.** Confirm the exact model identifiers in your environment. Pin explicit
model IDs, not bare aliases — an alias like "latest," "flagship," or a short family name can
resolve to different concrete models across providers and hosting platforms (a vendor's own
API vs. AWS Bedrock, Google Vertex, Azure, or a self-hosted gateway), so an alias is a weak
reproducibility claim. Pass the pinned ID into each subagent spawn (via its model
parameter/frontmatter). **Re-run the phase's completion test after any escalation.**

**Caveat — provider safety classifiers.** Some providers run safety classifiers that can
silently downgrade the model, refuse, or otherwise change behavior mid-session on security-
or biology-adjacent work. Where this happens it's often harmless cost-wise but
**nondeterministic** — so for reproducible behavior on security-critical phases, pin a
model/configuration that behaves deterministically for that content rather than one subject
to an automatic mid-session fallback.

### Persist the plan to the right layer

- **Ephemeral working state** (this task's plan, scratch diffs) → `./.work/PLAN.md`.
- **Durable decisions/conventions the team should keep** → project memory/rules (propose
  the exact lines; add them via whatever mechanism your tool uses).
- **Stable personal preferences** → user/global memory.

Keep entries short — most project rule files and memory indexes work best kept lean (a few
hundred lines at most). **Prune before you add.**

---

## 4. Execute

- Run **independent phases first / in parallel** where your tooling allows; respect
  dependencies otherwise.
- **After each phase, run its completion test.** Write the important intermediate artifact
  to `./.work/` before moving on.
- Delegate verbose work to subagents (if available); pull back only summaries.
- Memory is context, not enforcement: if a rule **must** hold (never touch prod, never
  commit secrets), enforce it with a **mechanical guardrail** — a pre-action hook, a
  deny-rule, a pre-commit/CI check, or a sandbox — not just a memory line, and say so
  explicitly. Whatever your tool offers for blocking an action *before it runs* is the
  right layer.

---

## 5. Adversarial review (separate subagent or fresh skeptical pass)

- Try to **break the result**: correctness, edge cases, security, and whether the
  completion tests *actually* prove "done."
- **Model:** top tier when the logic is critical or subtle; the high tier otherwise.
- Run it as a separate subagent if you have them, or as a deliberately fresh, skeptical
  pass otherwise — the point is independence from the mindset that wrote the code.
- Feed fixes back and **re-test**.

---

## 6. Re-plan

If evidence invalidates the path (a test can't pass, an assumption is wrong, scope shifts):
**stop.** Record what changed and why (`./.work/PLAN.md` for this task; project memory if
it's a lasting decision), re-plan the affected phases, and continue.

---

## 7. Close

Summarize:

- What was done.
- Per-phase completion-test results.
- Model per phase (and one-line why).
- Artifacts in `./.work/`.
- Proposed additions to project/user memory (give **exact lines**).
- Anything still open.

Flag which learnings are worth committing to memory vs. left as one-off — **skip anything
re-derivable from the code.**
