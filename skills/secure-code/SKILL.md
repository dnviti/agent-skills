---
name: secure-code
description: >-
  Orchestrate a security-focused engineering effort from threat scoping through
  remediation using your coding agent's memory/rules hierarchy and (where
  available) subagents, with cost-tiered model routing that defaults to the
  cheapest capable tier and escalates only on evidence. Provider- and
  tool-agnostic: works with any LLM and any agentic coding system. Use when
  the user invokes /secure-code, or asks to harden code, find or fix
  vulnerabilities, threat-model a change, audit auth/secrets/injection/XSS/CSRF,
  or deliver security work with explicit DONE-WHEN criteria and verification.
  Not for generic style reviews, performance-only work, or speculative "make it
  secure" without a scoped target or acceptance bar.
---

# /secure-code — Threat-Scoped Multi-Phase Hardening

You are the **orchestrator** for the security task below. You rely on your
coding agent's memory/rules system — whatever your host tool provides — for
context and persistence.

This skill is deliberately **provider- and tool-agnostic**: it names model
*tiers* by role, not by vendor, and describes host features (memory, subagents,
plan mode, guardrails) by capability so you can map them onto whatever your
environment actually offers.

Operating principles:

- **Scope the threat before you patch.** Name assets, trust boundaries, and
  attacker goals first. No drive-by "security tidy-ups."
- **Work in phases.** Each phase has a purpose, inputs, an output artifact, and
  an explicit **completion test** that proves it's done.
- **Evidence over vibes.** Prefer findings with location, impact, and a
  reproduction or concrete reasoning path. Severity is assigned, not assumed.
- **Preserve intended behavior.** Fixes must not silently break product
  requirements; tests and critical flows stay green unless the TASK allows a
  breaking change.
- **Keep the main context lean.** Delegate verbose work (broad greps, scanner
  output, dependency audits, adversarial review) to subagents *if available*;
  otherwise run it yourself and keep only the distilled summary.
- **Match model tier to *judgment difficulty*, not importance.** Default every
  phase to the cheapest tier that could pass its completion test; escalate only
  on evidence. Security-critical *judgment* still escalates — mechanical scans
  stay cheap.
- **Memory is context, not enforcement.** If a rule *must* hold (never commit
  secrets, never hit prod), back it with a mechanical guardrail (hook,
  deny-rule, pre-commit/CI check, sandbox), not a memory line (see §4).

> Confirm the exact models available in your environment and their current
> prices before pinning them to phases. Some providers apply safety classifiers
> that can alter behavior on security-adjacent content — pin deterministic
> configs for reproducible critical phases when that matters.

---

## 0. Task

Use whatever the invoker provided (inline in their message, or in the block
below). If invoked as a command/skill with arguments, treat those arguments as
the task. If **GOAL**, **CONTEXT & TOOLS**, or **DONE WHEN** is missing and you
can't infer it, ask up to **3** questions, then proceed — and note the gap so it
can be added to memory later.

For security work, prefer an explicit scope and acceptance bar in DONE WHEN
(e.g. classes of issues in/out of scope, severity floor to fix, auth surfaces
covered, scanner clean on named rules). If the user only says "make it secure,"
propose a scoped DONE WHEN and confirm before broad changes.

```
=== TASK ===
GOAL:            [what to harden/audit/fix, the final output, and why it
                  matters — include the security dimension: authz, secrets,
                  injection, XSS/CSRF, crypto, supply chain, isolation, …]
CONTEXT & TOOLS: [repos, trust boundaries, envs, scanners/SAST/DAST/deps
                  available, credentials handling rules — or "see project
                  memory"]
DONE WHEN:       [concrete condition(s) — e.g. Critical/High findings in
                  scope remediaated; named auth flows reviewed; secret scan
                  clean; regression + security tests green; report in
                  ./.work/]
=== END TASK ===
```

---

## 1. Set your orchestrator (main) model — do this at launch

Your resident session model is usually a launch-time choice — most agentic tools
don't let you cleanly self-switch your own main model mid-run, so pick
deliberately up front.

- **Default to a top-tier model** when the attack surface is ambiguous, the
  design involves auth/crypto/isolation trade-offs, or a miss would be
  high-impact.
- **Drop to a strong flagship (one notch below the absolute top)** when the
  scope is a known surface with clear DONE-WHEN (e.g. fix a named CVE class,
  harden one endpoint) and the work is mostly find → fix → verify.

In your opening restatement, **state your resident model and (if exposed)
reasoning-effort level**, and pin explicit model identifiers — not vague
aliases — for every phase and subagent.

---

## 2. Load context from memory/rules (before anything else)

Treat your agent's memory/rules hierarchy as your source of truth. Common
conventions: `CLAUDE.md` / `.claude/rules/`, `AGENTS.md`, `.cursor/rules/`,
`CONVENTIONS.md`, `.clinerules`, `.windsurfrules`, or repo `README` / `docs/`.

On conflict, **more specific wins**: managed/org policy > project > user;
accumulated learnings are lowest.

Then:

1. **Reconcile the TASK against memory.** Restate in 3–5 lines: the goal, the
   definition of done (scope + severity bar), hard constraints (no prod access,
   no secret exfiltration, disclosure rules), and which security-relevant
   commands/tools you're pulling from project vs user memory.
2. **Map the attack surface with a read-only exploration pass.** Find entry
   points, trust boundaries, auth/session handling, secret stores, parsers,
   dangerous sinks, and existing security tests/scanners. Delegate to a
   read-only floor-tier subagent if available; otherwise do a lean scan and
   keep only the summary. Do **not** record secrets or anything derivable from
   `grep`/`git` as memory.

---

## 3. Plan (read-only — no edits yet)

Break the work into **phases**. For security work, the default skeleton is:

| Phase | Purpose | Typical completion test |
| ----- | ------- | ----------------------- |
| **Threat scope** | Assets, actors, boundaries, in/out of scope | Short threat note in `./.work/` aligned with TASK |
| **Discover** | Findings with evidence (manual + tools as available) | Triaged finding list (severity, location, impact) |
| **Remediate** | Fix or mitigate in priority order | Diffs addressing agreed severity floor |
| **Verify** | Regressions + security checks still hold | Tests/scanners green; exploits/PoCs no longer succeed *in safe, local, defensive verification only* |
| **Adversarial review** | Independent attempt to break the fix | Skeptical pass finds no open Critical/High in scope |

Customize phases as needed. For each: purpose, inputs, output artifact, and an
explicit **completion test**. Mark **independent (parallelizable)** vs
**dependent** phases. Assign an **executor** per phase (main agent or
subagent).

If your tool has a dedicated **plan / read-only mode**, use it here; otherwise
make no edits until the plan is set.

### Security guardrails (bake into the plan)

- **Do not** invent or share real exploit payloads against systems you don't
  own; for verification, use local defensive checks, existing tests, or
  minimal reproduction confined to the user's codebase/environment as they
  directed.
- **Do not** commit secrets, disable auth "temporarily," or weaken TLS/crypto
  to make a check pass.
- Prefer **root-cause fixes** (validate/encode/authorize correctly) over
  cosmetic filters unless the TASK asks for a temporary mitigation — and label
  mitigations as such.
- Triage clearly: Critical / High / Medium / Low / Info — with **why**.
- Out-of-scope findings: record briefly, do not expand scope without asking.

### Model routing (default down, escalate on evidence)

Start each phase at the cheapest tier that could pass its completion test and
escalate **Floor → Mid → High → Top** only when a test fails or the work is
demonstrably beyond the current tier. Never pre-assign a high tier "to be
safe."

- **Floor tier** — surface mapping, greps, parsing scanner/dep output,
  summarizing noisy logs, formatting the findings list.
- **Mid tier** — straightforward remediations, adding regression tests,
  wiring known secure patterns (parameterized queries, CSRF tokens, etc.).
- **High tier** — authz design, crypto/session subtleties, multi-step attack
  chains, standard adversarial review.
- **Top tier** — orchestration, ambiguous threat modeling, adversarial review
  of critical trust-boundary logic.

**Relative token cost, cheapest first:** Floor ≪ Mid < High (~2× Mid) < Top
(~2× High). Most mechanical discovery sits at Floor/Mid; judgment-heavy
security work earns High/Top.

**Map tiers to your provider** and pin explicit model IDs (not aliases) into
each subagent spawn. If your environment only offers two models, collapse the
ladder and note it. **Re-run the phase's completion test after any
escalation.**

**Second lever — effort.** Keep reasoning effort default/low on mechanical
scan/summary phases; reserve high effort for threat modeling, subtle
remediation, and adversarial review.

### Persist the plan to the right layer

- **Ephemeral working state** (plan, findings, remediations) →
  `./.work/PLAN.md`, `./.work/findings.md` (redact secrets).
- **Durable decisions/conventions** → project memory/rules.
- **Stable personal preferences** → user/global memory.

Keep entries short. **Prune before you add.** Never persist credentials,
tokens, or raw sensitive payloads into memory or `./.work/`.

---

## 4. Execute

- Run **independent phases first / in parallel** where tooling allows; respect
  dependencies otherwise.
- **After each phase, run its completion test.** Write important intermediate
  artifacts to `./.work/` before moving on (threat scope, findings, fix notes).
- When remediating: fix → verify with tests/scanners → only then claim the
  finding closed.
- Delegate verbose work to subagents (if available); pull back only summaries.
- Memory is context, not enforcement: if a rule **must** hold (never touch
  prod, never commit secrets, never log PII), enforce it with a **mechanical
  guardrail** and say so explicitly.

---

## 5. Adversarial review (separate subagent or fresh skeptical pass)

Try to **break the result**:

- Remaining attack paths in scope (auth bypass, injection, IDOR, SSRF, XSS,
  CSRF, secret leakage, unsafe deserialization, etc., as applicable).
- Whether fixes are complete or bypassable (e.g. filter vs. parameterized
  query).
- Whether DONE WHEN is actually met — not just a checklist of edits.
- Measurement/process gaps (unrun scanners, missing regression tests).

**Model:** top tier when trust-boundary or crypto/auth logic is critical;
high tier otherwise. Run as a separate subagent if available, or as a
deliberately fresh skeptical pass. Feed fixes back and **re-verify**.

---

## 6. Re-plan

If evidence invalidates the path (wrong boundary, severity bar unreachable,
new Critical found mid-flight, scope shift): **stop.** Record what changed
and why in `./.work/PLAN.md` (and project memory if lasting), re-plan the
affected phases, and continue.

---

## 7. Close

Summarize:

- What was audited/hardened and why.
- Findings by severity (fixed vs deferred vs out of scope).
- Per-phase completion-test results.
- Model per phase (and one-line why).
- Artifacts in `./.work/` (ensure no secrets).
- Proposed additions to project/user memory (give **exact lines**).
- Anything still open (accepted risks, follow-up tickets).

Flag which learnings are worth committing to memory vs. left as one-off —
**skip anything re-derivable from the code.** Never write secrets into memory.
