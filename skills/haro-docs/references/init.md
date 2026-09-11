# Init (Haro Docs reference)
> **STOP — READ THIS FILE FULLY BEFORE ACTING.** This file is the normative workflow for `/haro-docs init`. Do not act, answer, edit, or call tools from memory: read every step below first. If in doubt at any point, re-read. The reference always wins over memory.
> **Ground rules (apply to every action in this file):** read `docroot` from `.haro-docs/project-profile.yaml` before operating — never guess it. Respect `language.response` (conversation) and `language.documentation` (doc content); if either is missing, ask the user first. Knowledge in `.haro-docs/knowledge/` is ground truth over scanned defaults (see `references/knowledge.md`).

## 5. Command `/haro-docs init <project description>`

Initialize the documentation structure for a project. Handles both fresh init and re-init of an already-initialized project.

### 5.1 Detect existing state

1. **Scan first** — same deep scan as SKILL.md (§3): README, source tree + tech stack, existing docs (inside and outside doc-root), `.haro-docs/` state (`project-profile.yaml`, `schema.yaml`, `schema_version`, knowledge file count, agents config).
2. **If `.haro-docs/` already exists** (re-init path) — show what exists (profile, schema_version, doc-root, docs file count, knowledge N files) and ask:
   - `[1] Override — rebuild from scratch` (fresh `.haro-docs` + doc tree from templates)
   - `[2] Migrate / Upgrade — update structure in place, keep content` (recommended when docs/knowledge already have value)
   - `[3] Cancel`
   
   Do NOT proceed without an explicit choice.
3. **If fresh** — continue with the required workflow below (§5.2).

### 5.2 Required workflow (fresh init)

1. **Scan the project** — read README, source code (tree structure, main technologies), and any existing docs to synthesize context: business domain, key features, code scale, technical constraints.
2. **Proactively ask clarifying questions** — ask the user, offering suggestions based on scan results:
   - Product/solution goals
   - Documentation audience (engineers, managers, customers...)
   - Scope and technical depth
   - Security/compliance requirements (if any)
   - Required output document types (BRD, PRD, SAD...)
   - **Reply language** — the language the agent uses in conversation: `en` or `vi`
   - **Documentation language** — the language of doc content: `en`, `vi`, or `vi-en` (definitions in references/authoring.md (§9))
3. **Ask for the doc-root** — the user chooses:
   - `docs/` (traditional documentation folder), or
   - `.haro-docs/docs/` (contained within the skill workspace)
3b. **Decide conventions (compact, 4–6 questions)** — Part A (fixed by skill: file naming, SSOT, status lifecycle, language ref, images, auto-file rule) is seeded from `templates/conventions.md` without asking; show it as read-only preview. Ask only Part B (project-specific), offering scan-based defaults:
   - Diagram tool: `mermaid | plantuml | drawio` (+ image fallback)
   - API spec format: `openapi-yaml | md-table | both`
   - Tone & depth: `high-level | balanced | deep-dive` (sync with `audience.technical_depth`)
   - RELEASED approver: single name/role, or per-domain approvers
   - Locked terms: product/brand terms with fixed spelling, or `(none)`
   - Priority deliverables: which of BRD/PRD/SAD/... first, or `(all)`
   
   Unanswered items use the stated defaults. Write `00-common/01-conventions.md` immediately (status RELEASED, single source MD-only — no YAML mirror).
4. **Confirm the outline** — present the folder tree + specific file list (including the decided conventions); wait for user approval. If the project already has non-conforming documentation, propose a reorganization plan (migration mapping table `old path → new path | keep / move / archive to 99-assets/_legacy/`) in this step.
5. **Initialize** — after approval:
   - Create `.haro-docs/project-profile.yaml` and `.haro-docs/schema.yaml` (from templates in the skill's `templates/` directory) with `schema_version: 1`, including the chosen `language.response` and `language.documentation`
   - Create `.haro-docs/agents.yaml` from `templates/agents.yaml` if it does not exist (never overwrite an existing one without asking)
   - Write `00-common/01-conventions.md` from `templates/conventions.md` with the Part B values from step 3b; create `02-references.md`, `03-abbreviations.md`, `04-glossary.md`, `05-traceability.md` as placeholders marked `auto-populated by generate — do not edit manually`
   - Create the folder tree per the outline, each folder gets a `README.md` describing its scope
   - Create a root overview README at the doc-root including the reading path

### 5.3 Re-init: Override vs Migrate

**Backup first (mandatory for both branches):** before touching anything, copy `.haro-docs/` + the doc-root tree to `.haro-docs.backup-<YYYYMMDD-HHmmss>/` and report the backup path in the reply. If backup fails, stop and ask the user.

| | Override (rebuild) | Migrate / Upgrade (in place) |
|---|---|---|
| `.haro-docs/project-profile.yaml` | Recreate from template (new `schema_version: 1`); re-ask language + doc-root | Keep + merge: preserve `language.*`, `docroot.path`, audience; only fill missing keys and bump `schema_version` |
| `.haro-docs/schema.yaml` | Recreate from template | Update in place: add missing folders/files, keep approved customizations; sync `meta.docroot` + `schema_version` |
| `.haro-docs/agents.yaml` | Recreate from template only if missing or user confirms | Keep user config; only merge missing keys (new agent ids, `default_reviewer`) |
| `knowledge/` | **Preserved by default** — copy back from backup; only drop if user explicitly says so | Fully preserved; `_index.md` rebuilt if inconsistent |
| Docs content | Fresh tree (user files gone from doc-root — still in backup) | Kept: never delete a user-written file; create only missing folders/files + missing `README.md` |
| `00-common/01-conventions.md` | Written from template with Part B from step 3b | If missing: seed Part A + ask Part B. If exists: keep Part B, refresh Part A only on user confirm |
| Off-schema files | N/A (fresh) | Propose mapping table; move to canonical path / keep / archive to `99-assets/_legacy/` only after per-row confirmation |

After either branch, report: `backup path | what was kept | what was rebuilt | what needs user review (mapping leftovers)`.

### 5.4 After init — next-step popup

When init (fresh, override, or migrate) completes, always show a next-step picker with **2–3 concrete smart suggestions** derived from the new state, e.g.:

- `generate 01-overview/01-problem-statement.md — foundational purpose is still empty`
- `config conventions — refine project-specific conventions if step 3b used defaults`
- `remember <seed fact from scan> — preserve stack/team facts`

Use the agent's question/picker tool when available, otherwise a numbered list. Wait for the user's pick; do NOT auto-run `generate` without confirmation.

### Examples

```
/haro-docs init E-commerce project with Next.js + PostgreSQL, team of 3 devs
/haro-docs init
```

With no description: still scan the current project first, then start asking from step 2 of §5.2.

