---
name: complex-work
description: >-
  Orchestrate a complex, multi-phase engineering task from planning through
  delivery using Claude Code's memory hierarchy and subagents, with cost-tiered
  model routing that defaults to the cheapest capable tier and escalates only on
  evidence. Use when the user invokes /complex-work, or asks to plan-and-execute
  a substantial task, break work into phases with explicit completion tests, run
  an orchestrator/subagent workflow, or wants adversarial review and re-planning
  built in. Not for quick one-off edits, single-file changes, or simple questions.
---

# /complex-work — Orchestrated Multi-Phase Execution

You are the **orchestrator** for the task below. You rely on Claude Code's memory
system for context and persistence. Operating principles:

- **Work in phases.** Each phase has a purpose, inputs, an output artifact, and an
  explicit **completion test** that proves it's done.
- **Keep the main context lean.** Delegate verbose or noisy work (research, big
  greps, test runs, log/doc scraping, adversarial review) to subagents and pull
  back only summaries.
- **Match model tier to *judgment difficulty*, not importance.** Default every
  phase to the cheapest tier that could plausibly pass its completion test;
  escalate only when a test actually fails or the work is demonstrably beyond the
  current tier. A phase can be critical to the project but mechanical to run — that
  is still a floor-tier job.
- **Memory is context, not enforcement.** If a rule *must* hold, back it with a
  hook, not a memory line (see §4).

> Model IDs and cost ratios in this skill are current as of authoring. Confirm live
> names with `/model` and check current pricing before relying on either.

---

## 0. Task

Use whatever the invoker provided (inline in their message, or in the block below).
If invoked as a slash command with arguments, treat those arguments as the task.
If **GOAL**, **CONTEXT & TOOLS**, or **DONE WHEN** is missing and you can't infer
it, ask up to **3** questions, then proceed — and note the gap so it can be added
to memory later.

```
=== TASK ===
GOAL:            [what to build/do, the final output, and why it matters]
CONTEXT & TOOLS: [repos, docs, credentials, environments, constraints — or "see project memory"]
DONE WHEN:       [the concrete condition(s) that mean this is finished]
=== END TASK ===
```

---

## 1. Set your orchestrator model (do this at launch)

The resident session model is a launch-time choice — you can't cleanly self-switch
your own main model mid-run.

- **Default to the top tier (currently Claude Fable 5 · `claude-fable-5`, alias
  `fable`)** for genuinely large, ambiguous, or multi-phase tasks. Long-running,
  investigate-before-acting, verify-your-own-work orchestration is its home turf.
- **Drop to the flagship (currently Claude Opus 4.8 · `claude-opus-4-8`)** when the
  task is well-scoped with an unambiguous DONE-WHEN and the plan is mostly dispatch
  — same capability class for well-scoped agentic work at roughly half the token
  cost, and the orchestrator session stays resident across every phase.

Launch with `claude --model <id>` (or `/model` in-session). In your opening
restatement, **state your resident model and effort level**, and pin explicit model
IDs — not aliases — for every phase and subagent.

---

## 2. Load context from memory (before anything else)

Treat the memory hierarchy as your source of truth:

- **User memory** (`~/.claude/CLAUDE.md`) — cross-project defaults.
- **Project memory** (`./CLAUDE.md` or `./.claude/CLAUDE.md`, plus `.claude/rules/`)
  — repo conventions, build/test commands, architecture.
- **Auto memory** — learnings accumulated across sessions.

On conflict, **more specific wins**: managed policy > project > user; auto memory is
lowest.

Then:

1. **Reconcile the TASK against memory.** Restate in 3–5 lines: the goal, the
   definition of done, hard constraints, and which key conventions/commands you're
   pulling from project vs user memory.
2. **Map the code with a read-only Explore subagent** (floor tier — see §3). Return
   only a summary. Do **not** record anything derivable from `grep`/`git` as memory.

---

## 3. Plan (plan mode — no edits yet)

- Break the work into **phases**. For each: purpose, inputs, output artifact, and an
  explicit **completion test** (a command/check that proves it's done).
- Mark **independent (parallelizable)** vs **dependent** phases and state the order.
- Assign an **executor** per phase: the main agent, or a subagent for token-heavy /
  noisy work and for adversarial review. Subagents may keep their own auto memory.

### Model routing (default down, escalate on evidence)

Start each phase at the cheapest tier that could pass its completion test and
escalate **Floor → Sonnet → Opus → Top** only when a test fails or the work is
demonstrably beyond the current tier. Never pre-assign a high tier "to be safe."
Reserve expensive tokens for genuinely hard reasoning plus adversarial review of
critical logic.

- **Floor tier — currently Claude Haiku 4.5 · `claude-haiku-4-5-20251001` (alias
  `haiku`).** High-volume, low-judgment, ideally read-only work: codebase mapping,
  big greps, scanning logs and verbose tool output, summarizing noisy results,
  mechanical text transforms, formatting, boilerplate scaffolding. **Run the
  read-only Explore subagent here** — this is the model whose whole purpose is
  keeping the resident session's context clean. Cheapest tokens; the throughput
  workhorse.
- **Coding default — currently Claude Sonnet 5 · `claude-sonnet-5`.** Standard
  implementation, most refactors, test writing, straightforward debugging. The large
  majority of real coding should land here.
- **Escalation — currently Claude Opus 4.8 · `claude-opus-4-8`.** Tricky debugging,
  security-sensitive logic, non-trivial or cross-cutting implementation, and standard
  adversarial review.
- **Top tier — currently Claude Fable 5 · `claude-fable-5` (alias `fable`).**
  Orchestration and the genuinely hardest work only: hard architecture, ambiguous or
  high-stakes design, and adversarial review of critical logic.

**Relative token cost, cheapest first:** Floor ≪ Sonnet < Opus (~2× Sonnet) < Top
(~2× Opus). Most work should sit at the bottom two tiers.

**Second lever — effort.** Keep `/effort` at default (or lower) on cheap mechanical
phases; reserve `high`/`xhigh` for hard-reasoning phases and the adversarial review.
Effort burns tokens too.

**Routing hygiene.** Confirm names with `/model`. Pin explicit model IDs, not aliases
— aliases resolve differently across the API, Bedrock, Vertex, and Foundry, so a bare
alias is a weak reproducibility claim. Pass the ID into each subagent spawn (agent
frontmatter `model:`, or the Agent/Task call's model param). **Re-run the phase's
completion test after any escalation.**

**Caveat for security-heavy phases.** The top tier's safety classifiers flag
cybersecurity- and biology-adjacent work and trigger an automatic fallback to a lower
model mid-session. This is harmless cost-wise (you don't pay top-tier rates for the
fallback) but it's **nondeterministic** — so for reproducible behavior on
security-critical phases, pin the flagship (Opus) rather than the top tier.

### Persist the plan to the right layer

- **Ephemeral working state** (this task's plan, scratch diffs) → `./.work/PLAN.md`.
- **Durable decisions/conventions the team should keep** → project memory (propose
  the exact lines; add via `/memory` or `#`).
- **Stable personal preferences** → user memory.

Keep entries short — project `CLAUDE.md` and the auto-memory index are capped (~200
lines). **Prune before you add.**

---

## 4. Execute

- Run **independent phases first / in parallel** where possible; respect dependencies
  otherwise.
- **After each phase, run its completion test.** Write the important intermediate
  artifact to `./.work/` before moving on.
- Delegate verbose work to subagents; pull back only summaries.
- Memory is context, not enforcement: if a rule **must** hold (never touch prod,
  never commit secrets), enforce it with a **PreToolUse hook**, not just a memory
  line — and say so explicitly.

---

## 5. Adversarial review (separate subagent)

- Try to **break the result**: correctness, edge cases, security, and whether the
  completion tests *actually* prove "done."
- **Model:** top tier when the logic is critical or subtle; flagship (Opus)
  otherwise.
- Feed fixes back and **re-test**.

---

## 6. Re-plan

If evidence invalidates the path (a test can't pass, an assumption is wrong, scope
shifts): **stop.** Record what changed and why (`./.work/PLAN.md` for this task;
project memory if it's a lasting decision), re-plan the affected phases, and continue.

---

## 7. Close

Summarize:

- What was done.
- Per-phase completion-test results.
- Model per phase (and one-line why).
- Artifacts in `./.work/`.
- Proposed additions to project/user memory (give **exact lines**).
- Anything still open.

Flag which learnings are worth committing to memory vs. left as one-off — **skip
anything re-derivable from the code.**
