---
name: optimize-code
description: >-
  Orchestrate a measured code-optimization effort from baseline through
  delivery using your coding agent's memory/rules hierarchy and (where
  available) subagents, with cost-tiered model routing that defaults to the
  cheapest capable tier and escalates only on evidence. Always seeks fewer
  lines of code while preserving every existing function and behavior.
  Provider- and tool-agnostic: works with any LLM and any agentic coding
  system. Use when the user invokes /optimize-code, or asks to speed up,
  reduce memory, cut latency/CPU, improve algorithmic complexity, shrink
  LOC without dropping features, profile hot paths, or optimize with
  explicit DONE-WHEN metrics and no-regression checks. Not for feature
  removal disguised as cleanup, speculative "make it faster" without a
  measurable target, or style-only formatting that does not reduce real
  code surface.
---

# /optimize-code — Measured Multi-Phase Optimization

You are the **orchestrator** for the optimization task below. You rely on your
coding agent's memory/rules system — whatever your host tool provides — for
context and persistence.

This skill is deliberately **provider- and tool-agnostic**: it names model
*tiers* by role, not by vendor, and describes host features (memory, subagents,
plan mode, guardrails, checkpoints) by capability so you can map them onto
whatever your environment actually offers. Wherever a concrete mechanism is
named, treat it as an example of a category — use your tool's equivalent.

Operating principles:

- **Measure before you change.** Establish a baseline, name the bottleneck, then
  optimize. No speculative rewrites.
- **Shrink LOC, keep all functions.** Every optimization pass must actively try
  to reduce lines of code (dedupe, collapse branches, delete dead paths, reuse
  helpers) **without removing or weakening any existing functionality**, public
  API, or required behavior. Fewer lines that drop a feature is a failure.
- **Work in phases.** Each phase has a purpose, inputs, an output artifact, and
  an explicit **completion test** that proves it's done.
- **Preserve correctness.** Behavior and tests must hold after every change;
  performance or LOC wins that break semantics are failures.
- **Keep every change reversible.** Snapshot before each optimize phase (a git
  commit/branch, or your tool's checkpointing) so a failed hypothesis rolls
  back cleanly instead of being patched forward (see §4).
- **Keep the main context lean.** Delegate verbose work (profiling runs, big
  greps, benchmark noise, adversarial review) to subagents *if available*;
  otherwise run it yourself and keep only the distilled summary.
- **Match model tier to *judgment difficulty*, not importance.** Default every
  phase to the cheapest tier that could pass its completion test; escalate only
  on evidence.
- **Memory is context, not enforcement.** If a rule *must* hold, back it with a
  mechanical guardrail (hook, deny-rule, pre-commit/CI check, sandbox), not a
  memory line (see §4).

> This skill names model tiers by role, not by vendor. Model lineups, aliases,
> and pricing change fast and differ across providers — confirm the exact
> models available in your environment and their current prices before pinning
> them to phases.

---

## 0. Task

Use whatever the invoker provided (inline in their message, or in the block
below). If invoked as a command/skill with arguments, treat those arguments as
the task. If **GOAL**, **CONTEXT & TOOLS**, or **DONE WHEN** is missing and you
can't infer it, ask up to **3** questions, then proceed — and note the gap so it
can be added to memory later.

For optimization work, prefer concrete metrics in DONE WHEN (e.g. p95 latency,
allocs/op, wall time for a named benchmark, asymptotic complexity of a hot
path, net LOC delta). Always include **LOC reduction with full feature
parity** as a standing goal unless the TASK explicitly forbids it. If the user
only says "make it faster," propose a measurable DONE WHEN (including LOC)
and confirm before heavy changes.

```
=== TASK ===
GOAL:            [what to optimize, the final output, and why it matters —
                  include the performance dimension: latency, throughput,
                  memory, CPU, binary size, algorithmic complexity, …
                  AND fewer lines with all functions preserved]
CONTEXT & TOOLS: [repos, hot paths, profilers/benchmarks available, envs,
                  constraints — or "see project memory"]
DONE WHEN:       [concrete measurable condition(s) — e.g. named bench under
                  X ms at p95; allocs/op ≤ Y; net LOC down vs baseline;
                  every prior function/behavior still present; existing
                  tests still green; no public API break]
=== END TASK ===
```

---

## 1. Set your orchestrator (main) model — do this at launch

Your resident session model is usually a launch-time choice — most agentic tools
don't let you cleanly self-switch your own main model mid-run, so pick
deliberately up front.

- **Default to a top-tier model** when the bottleneck is ambiguous, the
  optimization space is large, or correctness risk is high (concurrency,
  numerics, security-sensitive paths).
- **Drop to a strong flagship (one notch below the absolute top)** when the
  hot path and DONE-WHEN metric are already clear and the work is mostly
  measure → patch → re-bench.

Set the model however your tool allows — a CLI flag at launch (e.g.
`--model <id>`), an environment variable or config setting (e.g. a `model`
field in your tool's settings file), or an in-session command (e.g. `/model`).
In your opening restatement, **state your resident model and (if exposed)
reasoning-effort level**, and pin explicit model identifiers — not vague
aliases — for every phase and subagent.

---

## 2. Load context from memory/rules (before anything else)

Treat your agent's memory/rules hierarchy as your source of truth. The exact
files depend on your tool — common conventions include:

- **Project rules** — repo conventions, build/bench/test commands,
  architecture. E.g. `CLAUDE.md` / `.claude/CLAUDE.md` plus `.claude/rules/`
  (Claude Code, which also expands `@path` imports inside them), `AGENTS.md`
  (the emerging cross-tool convention), `.cursor/rules/` or `.cursorrules`
  (Cursor), a conventions file such as `CONVENTIONS.md` (Aider), `.clinerules`
  (Cline), `.windsurfrules` (Windsurf), or a repo `README` / `docs/`.
- **Personal project overlay** — an untracked, gitignored local rules file
  (e.g. `CLAUDE.local.md`) for machine-specific bench setups.
- **User / global defaults** — cross-project preferences (a home-directory
  rules file such as `~/.claude/CLAUDE.md`, global config).
- **Accumulated learnings** — any auto/persistent memory your tool keeps
  across sessions.

On conflict, **more specific wins**: managed/org policy > project > user;
accumulated learnings are lowest.

Then:

1. **Reconcile the TASK against memory.** Restate in 3–5 lines: the goal, the
   definition of done (including metrics), hard constraints (API stability,
   platform targets), and which build/bench/test commands you're pulling from
   project vs user memory.
2. **Map the code with a read-only exploration pass.** Find candidate hot
   paths, existing benchmarks, profilers, and related tests. Delegate to a
   read-only floor-tier subagent if available (e.g. Claude Code's built-in
   `Explore` agent); otherwise do a lean scan yourself — in your tool's
   read-only/plan mode if it has one — and keep only the summary. Do **not**
   record anything derivable from `grep`/`git` as memory.

---

## 3. Plan (read-only — no edits yet)

Break the work into **phases**. For optimization, the default skeleton is:

| Phase | Purpose | Typical completion test |
| ----- | ------- | ----------------------- |
| **Baseline** | Capture current metrics + LOC on the target scope | Reproducible bench/profile + LOC count under `./.work/` |
| **Locate** | Identify the real bottleneck and duplication (not a guess) | Named hot path / flamegraph / allocation site / redundant code with evidence |
| **Optimize** | Move the metric **and** cut lines without dropping functions | Diff + same bench improves toward DONE WHEN; net LOC ↓; feature inventory intact |
| **Verify** | Correctness + regression suite + feature parity | Tests green; no API/behavior/function loss |
| **Adversarial review** | Try to break the win | Skeptical pass finds no correctness, measurement, or silent feature-loss flaws |

Customize phases as needed. For each: purpose, inputs, output artifact, and an
explicit **completion test**. Mark **independent (parallelizable)** vs
**dependent** phases. Assign an **executor** per phase (main agent or
subagent).

If your tool has a dedicated **plan / read-only mode**, use it here (e.g.
Claude Code's plan mode — entered with `--permission-mode plan` or Shift+Tab —
guarantees reads only until the plan is approved); otherwise simply make no
edits until the plan is set.

### Optimization guardrails (bake into the plan)

- **Always prefer a shorter implementation** that preserves the full function
  set. Collapse duplication, dead code, and needless indirection; do **not**
  delete features, flags, error paths, or APIs to win on line count.
- Prefer **algorithmic / structural** wins over micro-tweaks unless the profile
  proves a micro-hotspot — structural wins often cut both cost and LOC.
- One hypothesis per optimize phase when possible — so you know what moved the
  needle (perf, LOC, or both).
- Never trade away correctness, security boundaries, required observability, or
  existing functionality for speed or brevity unless the TASK explicitly allows
  it.
- If a change improves a microbench but not the end-to-end DONE WHEN metric,
  treat it as inconclusive — do not claim success.
- If LOC cannot go down without losing a function, keep the lines, document why
  in `./.work/`, and still pursue the performance DONE WHEN.

### Model routing (default down, escalate on evidence)

Start each phase at the cheapest tier that could pass its completion test and
escalate **Floor → Mid → High → Top** only when a test fails or the work is
demonstrably beyond the current tier. Never pre-assign a high tier "to be
safe."

- **Floor tier — the cheapest, fastest small model.** Codebase mapping, greps,
  parsing profiler/bench output, summarizing noisy logs, mechanical
  formatting. **Run the read-only exploration pass here** — its whole purpose
  is keeping the resident session's context clean.
- **Mid tier — the standard coding model.** Standard optimizations,
  adding/adjusting benchmarks, most refactors that follow a clear profile.
- **High tier — a strong reasoning model.** Tricky concurrency/cache/layout
  issues, non-obvious algorithmic changes, security-sensitive hot paths,
  standard adversarial review.
- **Top tier — the most capable (and most expensive) model.** Orchestration,
  ambiguous architecture-level performance design, adversarial review of
  critical logic.

**Relative token cost, cheapest first:** Floor ≪ Mid < High (~2× Mid) < Top
(~2× High). Exact ratios differ by provider, but the shape holds. Most work
should sit at the bottom two tiers.

**Map tiers to your provider.** Every major lineup has this small → mid →
large → frontier shape; slot your available models into the four roles. For
example, Anthropic's Claude line maps as Haiku (floor) → Sonnet (mid) → Opus
(high) → its top reasoning model; OpenAI's GPT line, Google's Gemini line
(Flash-Lite/Flash/Pro tiers), and open-weight families (e.g. Llama, Qwen,
DeepSeek at increasing sizes) expose the same gradient. If your environment
only offers two models, collapse the ladder onto those two and note it.

**Routing hygiene.** Confirm the exact model identifiers in your environment.
Pin explicit model IDs, not bare aliases — an alias like "latest," "flagship,"
or a short family name can resolve to different concrete models across
providers and hosting platforms (a vendor's own API vs. AWS Bedrock, Google
Vertex, Azure, or a self-hosted gateway), so an alias is a weak
reproducibility claim. Pass the pinned ID into each subagent spawn through
whatever your tool provides — e.g. a `model` field in the subagent's
definition/frontmatter, a per-spawn model parameter, or an environment
override. **Re-run the phase's completion test after any escalation.**

**Second lever — effort.** If your model exposes a reasoning-effort /
thinking-budget control (e.g. Claude Code's `effort` setting and per-subagent
`effort` frontmatter, with levels from `low` to `max`), keep it at default or
low on mechanical measure/summarize phases and reserve high effort for
hard-reasoning optimize phases and the adversarial review. Effort burns tokens
too.

### Persist the plan to the right layer

- **Ephemeral working state** (this task's plan, baselines, bench diffs) →
  `./.work/PLAN.md` (and `./.work/baseline.*`, `./.work/after.*` as needed).
- **Durable decisions/conventions** (bench commands, perf budgets, hot-path
  ownership) → project memory/rules (propose the exact lines; add them via
  whatever mechanism your tool uses).
- **Stable personal preferences** → user/global memory.

Keep entries short — most project rule files and memory indexes work best kept
lean. **Prune before you add.**

---

## 4. Execute

- Run **independent phases first / in parallel** where your tooling allows;
  respect dependencies otherwise. If your tool supports background subagents
  or isolated worktrees (e.g. spawning a subagent with its own git worktree —
  or plain `git worktree` by hand), use them so parallel optimization
  experiments and long benchmark runs don't collide or block the main session.
- **Snapshot before each optimize phase.** A git commit/branch is the reliable
  layer. Tool checkpointing (e.g. Claude Code auto-checkpoints its own file
  edits and can rewind them) is a convenient extra, but note its limits — it
  typically does **not** track changes made by shell commands or scripts — so
  prefer git for anything a build step or codegen touches.
- **After each phase, run its completion test.** Write important intermediate
  artifacts to `./.work/` before moving on (baselines, profiles, before/after
  numbers and LOC).
- When optimizing: change → re-measure → compare to baseline (metrics **and**
  LOC) → confirm no function/behavior removed → only then proceed.
- Delegate verbose work to subagents (if available); pull back only summaries.
- Memory is context, not enforcement: if a rule **must** hold (never touch
  prod, never commit secrets, never weaken tests or benchmarks to "pass"),
  enforce it with a **mechanical guardrail** and say so explicitly. Whatever
  your tool offers for blocking an action *before it runs* is the right layer
  — e.g. a pre-action hook that rejects the call (Claude Code: a `PreToolUse`
  hook, where exit code 2 blocks the tool call), a permission deny-rule
  protecting the bench/test files from edits (Claude Code:
  `"deny": ["Edit(bench/**)"]` in `.claude/settings.json`), a pre-commit/CI
  check that re-runs the benchmark, or a sandbox.

---

## 5. Adversarial review (separate subagent or fresh skeptical pass)

Try to **break the result**:

- Correctness and edge cases (especially races, overflow, precision).
- **Silent feature loss** — any function, path, or API dropped or weakened
  solely to reduce LOC.
- Whether the benchmark actually exercises the claimed hot path.
- Measurement bias (warm vs cold, noisy CI, insufficient iterations).
- Whether DONE WHEN is truly met end-to-end — not only a microbench or a
  cosmetic LOC drop (e.g. stuffing many statements on one line).
- Hidden costs (memory, latency tails, binary size, maintainability).

**Model:** top tier when the logic is critical or subtle; high tier otherwise.
Run as a separate subagent if available, or as a deliberately fresh skeptical
pass — the point is independence from the mindset that wrote the change. Feed
fixes back and **re-test / re-bench**.

---

## 6. Re-plan

If evidence invalidates the path (metric won't move, assumption wrong, wrong
bottleneck, scope shifts): **stop.** Roll back to the last good snapshot
rather than patching forward. Record what changed and why in `./.work/PLAN.md`
(and project memory if lasting), re-plan the affected phases, and continue.

---

## 7. Close

Summarize:

- What was optimized and why.
- Baseline vs final metrics (same workload, same commands).
- Baseline vs final LOC (and confirmation that all prior functions remain).
- Per-phase completion-test results.
- Model per phase (and one-line why).
- Artifacts in `./.work/`.
- Proposed additions to project/user memory (give **exact lines**).
- Anything still open (follow-up wins you deliberately deferred).

Flag which learnings are worth committing to memory vs. left as one-off —
**skip anything re-derivable from the code.**
