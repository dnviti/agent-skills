# Contributing a skill

Thanks for adding to the catalog. Skills here follow the
[skills.sh](https://www.skills.sh) convention so they stay installable via the `skills` CLI.

## Add a skill

1. Create a directory under `skills/` named after your skill in `kebab-case`:

   ```
   skills/<skill-name>/SKILL.md
   ```

2. Write `SKILL.md` with YAML frontmatter and a body:

   ```markdown
   ---
   name: <skill-name>          # must match the directory name
   description: >-
     What the skill does AND when to use it. This is the text the agent matches
     against the task to decide whether to trigger the skill, so make the triggers
     explicit. Include a "Not for …" clause when the boundary matters.
   ---

   # <Skill Title>

   The instructions the agent follows once the skill is active.
   ```

3. Add a row to the **Available skills** table in [`README.md`](README.md).

## Conventions

- **`name` matches the directory.** The CLI and the agent both key off this.
- **Descriptions describe *when*, not just *what*.** The description is the trigger. Vague descriptions
  mean the skill never fires — or fires at the wrong time.
- **Keep skills self-contained.** A skill should not depend on files outside its own directory. If you
  need helpers, ship them inside `skills/<skill-name>/`.
- **One capability per skill.** Prefer several focused skills over one that tries to do everything.
- **Write in English** so the catalog stays consistent.

## Checklist before opening a PR

- [ ] `skills/<name>/SKILL.md` exists and `name:` matches the directory.
- [ ] `description:` states both what the skill does and when to use it.
- [ ] The skill is listed in the README table.
- [ ] You installed it locally (`npx skills add <your-fork> --skill <name>`) and confirmed the agent
      picks it up.
