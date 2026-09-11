# Config hub (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative workflow for `/haro-docs config`. Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults (see `references/knowledge.md`).

## 11. Command `/haro-docs config` — Config Hub (agents | conventions | language)

Central hub for skill configuration. Three branches, no `doc-root` here (changing doc-root is a heavy migrate — keep it in `init`).

### 11.1 Hub (no args)

``
/haro-docs config               → hub picker (read-only until a branch is chosen)
/haro-docs config agents        → straight into §11.2
/haro-docs config conventions   → straight into §11.3
/haro-docs config language      → straight into §11.4
``

Workflow:

1. Read state: `agents.yaml` (exists? N enabled, default?), `00-common/01-conventions.md` Part B values (6-row table or `missing`), `language.response/documentation` from `project-profile.yaml`.
2. Show picker (picker tool when available, otherwise numbered list), each row with a one-line status:
   - `1. agents — reviewer subagents (N enabled, default: <id>)`
   - `2. conventions — project-specific conventions (diagram, API format, tone, approver...)`
   - `3. language — reply + documentation language`
3. Enter the chosen branch. After a branch finishes, ask `back to hub / stop`. Unknown arg → `Unknown command 'config X'. Valid: agents, conventions, language.` then show the hub.

### 11.2 Branch: agents — Manage Reviewer Subagents

Manage the `.haro-docs/agents.yaml` configuration:

1. If the file is missing, create it from the skill's `templates/agents.yaml` and show its contents.
2. Otherwise list enabled/disabled agents as `id | role | enabled | default?`.
3. Offer operations (picker when available, otherwise numbered list): `add agent | edit role/prompt | enable / disable | set default_reviewer | reset from template (ask before overwriting custom prompts)`.
4. After any change, re-render the list and remind that `/haro-docs review` will offer these agents for selection. `init` never overwrites an existing `agents.yaml` without asking (it only merges missing keys) — see Section 5.3.

### 11.3 Branch: conventions — Edit Project-Specific Conventions

Edit Part B of `00-common/01-conventions.md` (Part A is fixed by the skill):

1. Show current Part B as table `item | value`; show Part A as 6 bullet titles only (full text on request).
2. Offer per-item edit (picker when available, otherwise numbered list): diagram tool | API spec format | tone & depth | RELEASED approver | locked terms | priority deliverables. Each item offers scan-based defaults.
3. `reset` re-seeds Part A from `templates/conventions.md` and **keeps Part B**. Confirm before any write.
4. After saving, ask `review RELEASED files affected by this change?` — only suggest the `review` command, never auto-edit RELEASED files. `init`/`migrate` never overwrite Part B without asking (see Section 5.3).

### 11.4 Branch: language — Reply + Documentation Language

1. Show current `language.response` + `language.documentation` from `.haro-docs/project-profile.yaml` (with `vi` vs `vi-en` definitions from references/authoring.md (§9)).
2. Offer change with `en | vi | vi-en` options as applicable; confirm before writing back to `project-profile.yaml`.
