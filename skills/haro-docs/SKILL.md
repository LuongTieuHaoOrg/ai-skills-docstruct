---
name: haro-docs
description: Manage project documentation structure using the Atomic Content Blocks model. Use when the user wants to initialize a documentation structure for a new project, organize/refactor existing documentation, aggregate complete documents (BRD, PRD, SAD, FSD...) from existing content blocks, critically review a problem/file via subagent reviewers, or manage project knowledge memory via remember/knowledge. Run /haro-docs with no args to scan the project and pick the next action. Before acting on any command, read its references/*.md file fully — never act from memory.
---

# Haro Docs — Documentation Structure Skill

## 1. Overview

`haro-docs` organizes project documentation as **Atomic Content Blocks**: every small section is a separate markdown file, written exactly once (Single Source of Truth), then flexibly assembled into complete documents (BRD, PRD, SAD, FSD...).

- **Standardization:** every project has an identical documentation structure.
- **Atomicity:** each block is an independent object — easy to track and version.
- **Flexibility:** output documents are just different "Views" over the same blocks (see `references/structure.md`).
- **Knowledge memory:** project facts live as small domain-scoped files under `.haro-docs/knowledge/` and are loaded selectively like RAG — no vector DB (see `references/knowledge.md`).

## 2. Workspace `.haro-docs/`

The skill stores all configuration and state in `.haro-docs/` at the project root:

```
.haro-docs/
├── project-profile.yaml   # Project profile: type, audience, doc-root, language settings, schema_version
├── schema.yaml            # Approved folder tree + aggregation matrix (+ schema_version)
├── agents.yaml            # Review subagent ("đệ tử") configuration — see references/review.md (§10)
├── status/                # Per-block status: DRAFT | UPDATING | RELEASED
├── knowledge/             # Project knowledge memory — selective RAG (see references/knowledge.md (§4))
│   ├── _index.md          # Lookup: file | domain | summary | tags
│   ├── business-*.md
│   ├── technical-*.md
│   ├── team-*.md
│   └── common-*.md
├── elicitation/           # Interim Q&A for generate (see references/generate.md (§6))
│   └── <sanitized-path>.md
└── reviews/               # Saved review reports: RR-YYYYMMDD-HHmmss-<slug>.md (see references/review.md (§10))
```

> **Important:** `project-profile.yaml` records the **doc-root** — the documentation location chosen by the user during `init`. Every command (`generate`, `remember`, `knowledge`) must read this config before operating. Never guess the doc-root.

> **Knowledge RAG:** Before any `init` or `generate` turn that needs project context, read `.haro-docs/knowledge/_index.md` (if it exists), then selectively load only the knowledge files whose domain/tags match the task. Use knowledge as ground truth when it conflicts with scanned defaults. See references/knowledge.md (§4).

> **Elicitation & Status:** `generate` stores interim Q&A in `.haro-docs/elicitation/` and marks each doc file `DRAFT | UPDATING | RELEASED` (frontmatter `status:` + `.haro-docs/status/`). See references/generate.md (§6).

> **Language settings:** `project-profile.yaml` also records the **reply language** (`language.response`) and the **documentation language** (`language.documentation`). These are the single source of truth for all communication and content decisions — see references/authoring.md (§9). If they are empty or missing, ask the user before running any command.

> ## MANDATORY ROUTING — READ BEFORE ACTING (no exceptions)
>
> This file is only the router. The normative workflow for each command lives in its reference file (table below).
>
> 1. Match the user's command to exactly one table row.
> 2. Read that reference file **fully, before any other tool call, edit, or answer** — the no-args dashboard (§3 below) is the only workflow that runs directly from this file.
> 3. If you notice you are about to act, answer, or create anything without the reference open, **STOP and read it first**. Acting from memory, habit, or a previous session instead of the reference is a workflow violation: **the reference always wins over memory**, even when you are confident. This applies equally to small/weak models — when in doubt, re-read.

## Command index

| Command | When to use | Read first (fully, before acting) |
|---------|-------------|-----------------------------------|
| `/haro-docs` (no args) | Scan project, show dashboard, pick next action | — (runs from §3 below) |
| `/haro-docs init <description>` | Initialize structure, or re-init / migrate if initialized | `references/init.md` |
| `/haro-docs generate` | Build next doc (role-adaptive Q&A) | `references/generate.md` + `references/authoring.md` when writing |
| `/haro-docs generate <file>` | Focus on a specific file | `references/generate.md` + `references/authoring.md` when writing |
| `/haro-docs review <topic\|file>` | Critically review a problem/file via subagent reviewer(s) | `references/review.md` |
| `/haro-docs remember <free text>` | Record knowledge (analyze → confirm → save) | `references/knowledge.md` |
| `/haro-docs knowledge` | List all knowledge files | `references/knowledge.md` |
| `/haro-docs config [agents\|conventions\|language]` | Manage config via hub picker | `references/config.md` |

## Authoring rules (summary — full text in `references/authoring.md`)

- Single Source of Truth: write once, assemble — never duplicate.
- Conversation uses `language.response`, doc content uses `language.documentation` (`en` | `vi` | `vi-en`); if missing, ask first.
- Write current state, never change history — no backward-compat phrasing (git owns versions).

## 3. Command `/haro-docs` (no args) — Project Scan + Status Dashboard + Action Picker

When the user runs `/haro-docs` with no arguments, or with arguments that do not match any configured command, do NOT execute a workflow. Instead run a **deep read-only scan** and show the dashboard + action picker:

1. **Deep scan (read-only)** —
   - If `.haro-docs/project-profile.yaml` and `.haro-docs/schema.yaml` exist: read `docroot`, `language.*`, `schema_version`; list the actual folder tree under doc-root (for each of `00-common` → `99-assets` show exists/missing, file count, and DRAFT/UPDATING/RELEASED breakdown from frontmatter `status:` + `.haro-docs/status/`).
   - If not initialized: show `Not initialized` and display the standard tree from references/structure.md (§8) as preview.
   - Check knowledge: if `.haro-docs/knowledge/_index.md` exists, show `Knowledge: N files` and the first 5 index rows; otherwise show `Knowledge: (empty)`.
   - Check agents config: if `.haro-docs/agents.yaml` exists, show `Agents: <ids> (default: <id>)`; otherwise show `Agents: (default inline critic)`.
   - Scan the repo lightly: README (business domain, key features), top-level source tree + tech stack signals (package.json / requirements / go.mod / pom.xml / Cargo.toml...), code scale estimate, docs files lying outside doc-root (if any).
   - Synthesize a **Project Note**: 5–8 lines on current state — initialized?, doc-root, docs coverage (% RELEASED), biggest gaps (top-3 empty folders/files), tech stack, knowledge depth.
2. **Show command summary:**

   | Command | When to use | Example |
   |---------|-------------|---------|
   | `/haro-docs init <description>` | Initialize structure (12 folders), or re-init / migrate if already initialized | `/haro-docs init E-commerce Next.js + PostgreSQL` |
   | `/haro-docs generate` | Build next doc in order (role-adaptive Q&A) | `/haro-docs generate` |
   | `/haro-docs generate <file>` | Focus on a specific file | `/haro-docs generate 02-business/01-value-prop.md` |
   | `/haro-docs review <topic\|file>` | Critically review a problem/file via subagent reviewer(s) | `/haro-docs review Should we use microservices?` |
   | `/haro-docs remember <free text>` | Record knowledge (analyze → confirm → save) | `/haro-docs remember STID is my company` |
   | `/haro-docs knowledge` | List all knowledge files | `/haro-docs knowledge` |
   | `/haro-docs config [agents\|conventions\|language]` | Manage skill config via hub picker (subagents, conventions, language) | `/haro-docs config` |

3. **Show Aggregation Matrix (compact)** — BRD/PRD/SAD/FSD source folders from references/structure.md (§7).
4. **Action picker (popup)** — after the dashboard, always ask the user what to do next (use the agent's question/picker tool when available, otherwise a numbered list). Pre-suggest **2–3 smart recommendations** based on the scan, e.g.:
   - Not initialized → recommend `init`.
   - `01-overview` / `02-business` empty → recommend `generate <that file>`.
   - Many files `RELEASED` but no recent review → recommend `review <topic>`.
   - Knowledge empty → recommend `remember <seed facts>`.
   The user may pick a suggestion or name any other command. Once picked, follow the MANDATORY ROUTING above: read that command's reference file fully before acting. Do NOT auto-run side effects without that command's normal confirmations.
5. **Do not create or modify any file.** If the first token is unknown (e.g. `/haro-docs foo`), prefix the dashboard with `Unknown command 'foo'. Valid: init, generate, review, remember, knowledge, config.` and suggest the closest match. Also handle `help`, `--help`, `-h` as aliases for this dashboard. Matching is case-insensitive, trim whitespace.

