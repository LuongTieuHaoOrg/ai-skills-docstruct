# haro-skills

A collection of agent skills built by Haro. Each skill lives in its own folder under `skills/`.

| Skill | What it does | Path |
|-------|--------------|------|
| Haro Docs | Manage project documentation (PLM docs for software) using the Atomic Content Blocks model: init structure, generate docs, critical review via subagents, knowledge memory | `skills/haro-docs/` |

## Adding a new skill

1. Create `skills/<skill-name>/` with a `SKILL.md` (frontmatter `name`, `description`).
2. Keep skill assets (e.g. `templates/`) inside the skill folder.
3. Add one row to the table above.
