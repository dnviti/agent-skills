# agent-skills

A catalog of **Agent Skills** for [Claude Code](https://code.claude.com/docs/en/skills) and other
compatible AI coding agents. Every skill in this repo is a self-contained `SKILL.md` file that teaches
an agent a reusable, procedural capability — install one and the agent picks it up automatically.

This repository follows the [skills.sh](https://www.skills.sh) catalog convention, so any skill here can
be installed with the `skills` CLI, or dropped into your agent's skills directory by hand.

## Available skills

| Skill | Description |
| ----- | ----------- |
| [`complex-work`](skills/complex-work/SKILL.md) | Orchestrate a complex, multi-phase engineering task from planning through delivery using Claude Code's memory hierarchy and subagents, with cost-tiered model routing that defaults to the cheapest capable tier and escalates only on evidence. |

## Install

### Option 1 — the `skills` CLI (recommended)

Install every skill in this catalog into the current project:

```bash
npx skills add dnviti/agent-skills
```

Or install a single skill by name:

```bash
npx skills add dnviti/agent-skills --skill complex-work
```

Useful flags:

- `--list` — list the skills in this repo without installing anything.
- `-g` / `--global` — install into your home directory (`~/.claude/skills/`) so the skill is available
  in **every** project, instead of the default project-local `./.claude/skills/`.
- `-a <agent>` — target a specific agent (e.g. `claude-code`, `cursor`) when you use more than one.

Claude Code discovers `SKILL.md` files automatically the next time you start a session. Confirm a skill
loaded with `/skills` (or by asking the agent what skills it has).

### Option 2 — manual install

Each skill is just a directory with a `SKILL.md` inside. Copy the one you want into your agent's skills
directory:

```bash
# Project-local (committed with your repo, shared with your team)
mkdir -p .claude/skills
cp -r skills/complex-work .claude/skills/

# Or global (available in every project on your machine)
mkdir -p ~/.claude/skills
cp -r skills/complex-work ~/.claude/skills/
```

Restart your agent session and the skill is live.

## Repository layout

```
agent-skills/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
└── skills/
    └── <skill-name>/
        └── SKILL.md
```

The `skills/<name>/SKILL.md` layout is exactly what the `skills` CLI expects — it performs a depth-2
catalog walk, so both flat (`skills/<name>/SKILL.md`) and categorized
(`skills/<category>/<name>/SKILL.md`) layouts are discovered automatically.

## Anatomy of a skill

A skill is a Markdown file with YAML frontmatter. The frontmatter is what the agent reads to decide
*when* to reach for the skill; the body is the procedure it follows once triggered.

```markdown
---
name: my-skill
description: >-
  One or two sentences on what the skill does and, crucially, WHEN to use it.
  The agent matches this text against the task, so be explicit about triggers.
---

# My Skill

Step-by-step instructions the agent should follow when this skill is active.
```

**Frontmatter fields**

| Field | Required | Notes |
| ----- | -------- | ----- |
| `name` | yes | Kebab-case; must match the directory name. |
| `description` | yes | What the skill does **and when to use it** — the trigger text the agent matches against. |

## Contributing

New skills are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for the conventions and the checklist.

## License

Released under the [MIT License](LICENSE).
