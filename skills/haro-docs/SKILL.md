---
name: haro-docs
description: Manage project documentation structure using the Atomic Content Blocks model. Use when the user wants to initialize a documentation structure for a new project, organize/refactor existing documentation, aggregate complete documents (BRD, PRD, SAD, FSD...) from existing content blocks, critically review a problem/file via subagent reviewers, or manage project knowledge memory via remember/knowledge. Run /haro-docs with no args to scan the project and pick the next action.
---

# Haro Docs — Documentation Structure Skill

## 1. Overview

`haro-docs` organizes project documentation as **Atomic Content Blocks**: every small section of documentation is a separate markdown file, written exactly once (Single Source of Truth), then flexibly assembled into complete documents (BRD, PRD, SAD, FSD...).

Key strengths:

- **Standardization:** every project has an identical documentation structure, making quality comparable.
- **Atomicity:** each block is an independent object — easy to assign, track status, and version.
- **Flexibility:** output documents are just different "Views" aggregated from the same source blocks via the **Aggregation Matrix** (Section 7).
- **Knowledge memory:** project facts (business, technical, team, conventions) are stored as small domain-scoped files under `.haro-docs/knowledge/` and selectively loaded like RAG — no vector DB needed.

## 1.1 Knowledge as Selective RAG (no vector DB)

Because vector search is not available, knowledge is organized for **filename + index** retrieval:

- Each fact is stored in a file named by domain: `business-*.md`, `technical-*.md`, `team-*.md`, `common-*.md`, `security-*.md`, etc. The agent chooses the name that best matches the content so it can be found without loading all memory.
- `.haro-docs/knowledge/_index.md` is the single lookup table: for each file, one row with `file | domain | one-line summary | tags`. Agents read **only `_index.md`** (small) to decide which files to load for the current task, then load only those files. Never load the entire `knowledge/` directory by default.
- Use `_index.md` before every `init` or `generate` turn that needs project context: read `_index.md`, pick rows whose domain/tags match the task (e.g. business for BRD, technical/architecture for SAD), load only those files. Knowledge overrides scanned defaults when they conflict.

## 2. Workspace `.haro-docs/`

The skill stores all configuration and state in `.haro-docs/` at the project root:

```
.haro-docs/
├── project-profile.yaml   # Project profile: type, audience, doc-root, language settings, schema_version
├── schema.yaml            # Approved folder tree + aggregation matrix (+ schema_version)
├── agents.yaml            # Review subagent ("đệ tử") configuration — see Section 10
├── status/                # Per-block status: DRAFT | UPDATING | RELEASED
├── knowledge/             # Project knowledge memory — selective RAG (see Section 4)
│   ├── _index.md          # Lookup: file | domain | summary | tags
│   ├── business-*.md
│   ├── technical-*.md
│   ├── team-*.md
│   └── common-*.md
├── elicitation/           # Interim Q&A for generate (see Section 6)
│   └── <sanitized-path>.md
└── reviews/               # Saved review reports: RR-YYYYMMDD-HHmmss-<slug>.md (see Section 10)
```

> **Important:** `project-profile.yaml` records the **doc-root** — the documentation location chosen by the user during `init`. Every command (`generate`, `remember`, `knowledge`) must read this config before operating. Never guess the doc-root.

> **Knowledge RAG:** Before any `init` or `generate` turn that needs project context, read `.haro-docs/knowledge/_index.md` (if it exists), then selectively load only the knowledge files whose domain/tags match the task. Use knowledge as ground truth when it conflicts with scanned defaults. See Section 4.

> **Elicitation & Status:** `generate` stores interim Q&A in `.haro-docs/elicitation/` and marks each doc file `DRAFT | UPDATING | RELEASED` (frontmatter `status:` + `.haro-docs/status/`). See Section 6.

> **Language settings:** `project-profile.yaml` also records the **reply language** (`language.response`) and the **documentation language** (`language.documentation`). These are the single source of truth for all communication and content decisions — see Section 9. If they are empty or missing, ask the user before running any command.

## 3. Command `/haro-docs` (no args) — Project Scan + Status Dashboard + Action Picker

When the user runs `/haro-docs` with no arguments, or with arguments that do not match any configured command, do NOT execute a workflow. Instead run a **deep read-only scan** and show the dashboard + action picker:

1. **Deep scan (read-only)** —
   - If `.haro-docs/project-profile.yaml` and `.haro-docs/schema.yaml` exist: read `docroot`, `language.*`, `schema_version`; list the actual folder tree under doc-root (for each of `00-common` → `99-assets` show exists/missing, file count, and DRAFT/UPDATING/RELEASED breakdown from frontmatter `status:` + `.haro-docs/status/`).
   - If not initialized: show `Not initialized` and display the standard tree from Section 8 as preview.
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

3. **Show Aggregation Matrix (compact)** — BRD/PRD/SAD/FSD source folders from Section 7.
4. **Action picker (popup)** — after the dashboard, always ask the user what to do next (use the agent's question/picker tool when available, otherwise a numbered list). Pre-suggest **2–3 smart recommendations** based on the scan, e.g.:
   - Not initialized → recommend `init`.
   - `01-overview` / `02-business` empty → recommend `generate <that file>`.
   - Many files `RELEASED` but no recent review → recommend `review <topic>`.
   - Knowledge empty → recommend `remember <seed facts>`.
   The user may pick a suggestion or name any other command. Do NOT auto-run the picked command's side effects without the normal confirmations of that command.
5. **Do not create or modify any file.** If the first token is unknown (e.g. `/haro-docs foo`), prefix the dashboard with `Unknown command 'foo'. Valid: init, generate, review, remember, knowledge, config.` and suggest the closest match. Also handle `help`, `--help`, `-h` as aliases for this dashboard. Matching is case-insensitive, trim whitespace.

## 4. Command `/haro-docs remember / knowledge` — Project Knowledge Memory (Selective RAG)

Knowledge files are project facts the agent must remember and follow. They are stored as small domain-scoped files so they can be loaded selectively without reading all memory.

### Storage

```
.haro-docs/knowledge/
├── _index.md          # file | domain | summary | tags — the only file read by default
├── business-*.md
├── technical-*.md
├── team-*.md
├── common-*.md
└── security-*.md
```

`_index.md` format (one row per knowledge file):

| file | domain | summary | tags |
|------|--------|---------|------|
| business-stid-team.md | business | STID is company, PM is Hao, members Hao/Vu/Dai | stid, team, pm |
| technical-stack.md | technical | Stack: Next.js + PostgreSQL | nextjs, postgres |

Domains: `business`, `technical`, `team`, `common`, `security`, `quality`, `operations`, `guides` — pick the closest. The agent chooses the filename that best matches the content (e.g. `business-stid-team.md` for team facts) so it can be found without loading all memory. Keep each file focused (1-3 facts); split a user message that contains multiple domains into multiple files.

### Selective loading (RAG without vector DB)

Before any `init` or `generate` turn that needs project context:

1. Read `knowledge/_index.md` if it exists (small, always).
2. Pick only rows whose `domain`/`tags`/`summary` match the current task (e.g. `business` for BRD/Proposal, `technical`+`architecture` for SAD/FSD, `team` for any task mentioning people).
3. Load only those matched knowledge files. Do NOT load the whole `knowledge/` directory.

Knowledge is ground truth: when a knowledge file conflicts with scanned code/README, prefer knowledge.

### Subcommands

| Subcommand | Behaviour |
|------------|-----------|
| `remember <free text>` | **Analyze → confirm → save.** Parse the free text, extract distinct facts, rewrite each into a canonical sentence (current-state style per Section 9, no history phrasing), choose a domain filename for each, and show a preview table `file | canonical content` + updated `_index` rows. Then ask `Confirm? (yes / edit <corrections>)`. Only on `yes` (or after applying `edit`) write the files and update `_index.md`. If the user says `edit`, re-render the preview with corrections and ask again. `knowledge <free text>` is an alias for this subcommand. No backward compatibility is kept. |
| `knowledge` | **List all knowledge** (no args). Read `_index.md` and render the table plus file counts. If `_index.md` missing but files exist, rebuild it from filenames. Read-only. |

Unknown subcommand for this group → `Unknown command 'X'. Valid: remember <free text>, knowledge.` then show this table. `help`/`--help`/`-h` also show it.

### Examples

```
/haro-docs remember STID is my company, I am PM, members are Hao (me), Vu (Backend) and Dai (Frontend)
/haro-docs knowledge
```

## 5. Command `/haro-docs init <project description>`

Initialize the documentation structure for a project. Handles both fresh init and re-init of an already-initialized project.

### 5.1 Detect existing state

1. **Scan first** — same deep scan as Section 3: README, source tree + tech stack, existing docs (inside and outside doc-root), `.haro-docs/` state (`project-profile.yaml`, `schema.yaml`, `schema_version`, knowledge file count, agents config).
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
   - **Documentation language** — the language of doc content: `en`, `vi`, or `vi-en` (definitions in Section 9)
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

## 6. Command `/haro-docs generate` / `generate <file>` — Build Docs in Order (Role-Adaptive)

Build documentation flexibly based on current state + your role. Two modes:

- `/haro-docs generate` — propose 2–3 next files to build, based on reality, not rigid order. Skips all `00-common` files — `01-conventions.md` is decided during `init` (§5.2 step 3b) and `02-references, 03-abbreviations, 04-glossary, 05-traceability` are auto-populated; start writing at `01-overview`.
- `/haro-docs generate <file>` — focus on a specific file (e.g. `02-business/01-value-prop.md` or `10-deliverables/01-BRD.md`) to create or adjust it. If the file is `RELEASED`, warn and ask to update.

### Ordered index

Canonical order is `00-common → 01-overview → 02-business → 03-features → 04-architecture → 05-security → 06-implementation → 07-quality → 08-operations → 09-guides → 10-deliverables → 99-assets` as defined in `schema.yaml` and Section 8. Within each folder, files are ordered by numeric prefix. This order is the **reference**, not a rigid gate: use it to understand what is prerequisite for what, but do not block flexibly. If all files are `RELEASED`, reply `All done — every file is RELEASED.`.

### Elicitation storage

Interim Q&A is stored in `.haro-docs/elicitation/<sanitized-path>.md` (e.g. `02-business-01-value-prop.md`) to avoid context overload. It is read on demand during `generate` and not loaded by default. After the target file is completed, ask the user whether to keep or delete the elicitation file.

### Status

Each doc file carries `status:` in frontmatter and a mirror in `.haro-docs/status/<path>`:

- `DRAFT` — brand-new file, first version.
- `UPDATING` — has reference content and is being edited.
- `RELEASED` — completed, may be referenced by later files, not edited. If `generate <file>` targets a `RELEASED` file, warn `File is RELEASED` and ask `Update? (yes/no)`. On `yes`, set status to `UPDATING` before editing.

No backward compatibility: every creation or adjustment is treated as the **first version** per Section 9.

### Workflow

1. **Selective RAG** — read `knowledge/_index.md` (if exists), pick rows whose domain/tags match the target file (`business` for BRD/Proposal, `technical`+`architecture` for SAD/FSD, `team` for people), load only those knowledge files.
2. **Ask who you are & propose next file based on reality:**
   - Ask first: `Who are you in this project? What is your role? (customer / sales / BA / dev / PM) — what do you know best about?` Save answer to `knowledge/team-role.md` if not yet stored (or confirm `Still <role>?`), and also to `elicitation/<target>.md`.
    - Scan the current docs reality: which files are missing/empty (`DRAFT`), which are `UPDATING`, which are `RELEASED`, which `change-requests/CR-*.md` are new. Use the canonical order as reference to understand prerequisites (e.g. `01-overview` and `02-business` are prerequisites for `04-architecture`), but **do not enforce rigidly** — understand flexibly what is actually needed. Skip the whole `00-common` folder (`01-conventions.md` is decided at init, `02-05` are auto); start writing at `01-overview`, not `00-common`.
   - If `<file>` was given: treat it as the user's preference, but if it depends on an unfinished prerequisite (e.g. targeting `04-architecture` while `01-overview/03-goals.md` is still empty), explain why the prerequisite matters and ask `Do you want to continue with this file or switch to the prerequisite?`.
   - If no `<file>`: propose **2–3 candidates** for the next file, each with a one-line reason (e.g. `1. 01-overview/01-problem-statement.md — foundational purpose is still empty; 2. 02-business/01-value-prop.md — business value needed before architecture`). Let the user pick. If the user picks none, they can specify another file.
   - Once a target is chosen/confirmed, read its existing content if any (`UPDATING` case) and nearby files in the same folder + glossary for context. Note current status (`DRAFT` if new, `UPDATING` if exists).
3. **Role-adaptive elicitation (flexible, not rigid):**
   - Generate **~5 tailored questions** based on `role + file domain + current reality` (e.g. for `02-business/*` with customer → market/pain/SLA/metrics; for `04-architecture/*` with customer → flows/business rules only, with dev → endpoints/schemas/NFR/diagrams). **Never ask a customer about code**, but otherwise adapt to what is actually missing — do not apply a hard rule like "purpose before technical" to every case.
   - Append Q&A to the elicitation file.
   - Analyze: check which required sections of the target file (per schema) are still missing. If incomplete, ask another 3–5 follow-up questions, append to elicitation file, repeat until sufficient.
4. **Propose outline** — list sections for the target file, annotating source (which block/folder and which knowledge files). Wait for outline approval.
5. **Write** — create/update the target file in its canonical folder (kebab-case + numeric prefix, Section 9) using current-state style (Section 9, no backward). Follow `00-common/01-conventions.md` Part B (diagram tool, API spec format, tone & depth, locked terms); if conventions lack guidance for this file, use the sensible default and note it in one line without editing conventions. Update `status:` frontmatter to `DRAFT` or `UPDATING` as appropriate. If knowledge facts were introduced, also update `knowledge/` if needed. Record `related_paths` if any.
6. **Auto-update 00-common** — immediately after writing, scan the new content for glossary terms, abbreviations, and references not yet in `00-common/`. Auto-append them to the corresponding file without requiring another command:
   - New term → append to `00-common/04-glossary.md` with one-line definition inferred from context; if `vi-en` mode, add English gloss.
   - New abbreviation → append to `00-common/03-abbreviations.md`.
   - New external reference (link, doc, standard) → append to `00-common/02-references.md`.
   No confirmation needed; just note `Auto-updated 00-common: +2 terms, +1 abbreviation.` in the reply. Deduplicate before appending and respect `status: RELEASED` — still auto-append even to RELEASED `00-common` files (they are living references).
7. **Elicitation cleanup** — ask `Keep elicitation file for reference or delete? (keep / delete)`. Act accordingly.
8. **Confirm & mark status** — ask `Mark this file as? (DRAFT / UPDATING / RELEASED)`. Default is `RELEASED` if the user says the file is done. Update frontmatter and `.haro-docs/status/<path>` accordingly. `RELEASED` files become referenceable by later `generate` runs.
9. **Next-step popup** — after marking status, always ask what to do next (picker tool when available, otherwise numbered list). Propose 2–3 concrete candidates based on the fresh reality scan (prerequisites + gaps), each with a one-line reason, plus `review <just-finished file>` and `stop`. Example: `1. generate 02-business/01-value-prop.md — business value needed before architecture; 2. review 01-overview/01-problem-statement.md — just RELEASED, worth a critic pass; 3. stop`. Wait for the pick; do NOT auto-run.

### Examples

```
/haro-docs generate
/haro-docs generate 02-business/01-value-prop.md
/haro-docs generate 10-deliverables/01-BRD.md
```

## 7. Aggregation Matrix

| Document | Assembled from |
|----------|----------------------------|
| **BRD** | 01-overview + 02-business |
| **PRD** | 01-overview + 03-features + 02-business + 07-quality |
| **SAD** | 04-architecture + 05-security + 06-implementation + 07-quality + 08-operations |
| **FSD** | 03-features + 04-architecture |
| **SRD** | 05-security + 03-features |
| **Proposal** | 01-overview + 02-business |
| **Test Plan** | 07-quality + 03-features + 05-security |
| **Runbook** | 08-operations + 06-implementation + 04-architecture |
| **User/Admin Guide** | 09-guides + 03-features + 01-overview |

## 8. Standard structure (single)

Single structure for all projects:

```
docs/                        # or .haro-docs/docs/ depending on user-chosen doc-root
├── 00-common/               # 01-conventions.md, 02-references.md, 03-abbreviations.md, 04-glossary.md, 05-traceability.md (01 manual, 02-05 auto)
├── 01-overview/             # 01-problem-statement.md, 02-vision.md, 03-goals.md, 04-scope.md, 05-stakeholders.md, 06-constraints.md, 07-roadmap.md (Version|Goal|Target|Status)
├── 02-business/             # 01-value-proposition.md, 02-market-analysis.md, 03-business-model.md, 04-pricing.md, 05-sla.md, 06-risk-legal.md, use-cases/, change-requests/CR-*.md
├── 03-features/             # 01-feature-catalog.md, 04-dependencies.md, features/<feature>/ 01-overview.md, 02-user-stories.md, 03-acceptance-criteria.md
├── 04-architecture/         # 01-system-overview.md, 02-components.md, 03-data-flow.md, 04-api-spec.md, 05-data-model.md, 06-tech-stack.md, 07-decisions-adr.md
├── 05-security/             # 01-threat-model.md, 02-defense.md, 03-compliance.md, 04-privacy.md, 05-authz.md
├── 06-implementation/       # 01-deployment.md, 02-configuration.md, 03-integration.md, 04-migration.md, 05-development-guide.md
├── 07-quality/              # 01-test-strategy.md, 02-test-plan.md, 03-test-cases.md, 04-uat.md, 05-quality-metrics.md
├── 08-operations/           # 01-runbook.md, 02-incident-response.md, 03-dr-bcp.md, 04-monitoring.md, 05-support.md
├── 09-guides/               # 01-user-guide.md, 02-admin-guide.md, 03-training.md, 04-faq.md, 05-onboarding.md
├── 10-deliverables/         # 01-BRD.md, 02-PRD.md, 03-SAD.md, 04-FSD.md, 05-SRD.md, 06-Proposal.md, 07-TestPlan.md, 08-Runbook.md, 09-UserAdminGuide.md (placeholders with distinct headings + Ref links, not identical)
└── 99-assets/               # Images, diagrams, templates
```

`init` creates all folders/files above; each `10-deliverables/*.md` is a placeholder with its own headings plus `Ref: ../01-overview/...` links — not identical templates. In `00-common`, `01-conventions.md` is decided during `init` (§5.2 step 3b, 2 layers: fixed Part A + project-specific Part B, MD-only single source) while `02-references, 03-abbreviations, 04-glossary, 05-traceability` are living references auto-populated by `generate` from placeholders marked `auto-populated by generate — do not edit manually`; `05-traceability.md` format is a table `deliverable | source blocks | block status | knowledge refs`. `01-overview` is the starting point for writing (not `00-common`).

### Distinguishing `03-features` from `02-business/use-cases/`

These two folders cause the most confusion — distinguish by audience and detail level:

| | `03-features` | `02-business/use-cases/` |
|---|---------------|-------------------------------|
| Content | User stories + acceptance criteria + technical references | End-to-end interaction flows: main path, all alternative/exception paths |
| Audience | Developers, testers | Customers, BAs, test case writers |
| Language | Technical (schema/endpoint references allowed) | Pure business language, no technical detail |
| SSOT rule | Stories cross-link to use cases for context | Use cases contain NO technical detail |

One use case is typically decomposed into multiple user stories; each story links back to its source use case instead of retelling the flow.

Folders not yet needed may stay with their `README.md` and a short `> Out of scope for this project` note.

## 9. Authoring Rules

1. **Single Source of Truth:** each piece of content is written once in one file only; aggregated documents only assemble content, never duplicate it.
2. **Language configuration** — read from `project-profile.yaml` (`language.response`, `language.documentation`):
   - **Conversation replies** (clarifying questions, elicitation Q&A, outline proposals...) use `language.response`.
   - **Documentation content** uses `language.documentation`, defined as:

     | Mode | Definition | Example |
     |------|------------|---------|
     | `en` | Pure English | — |
     | `vi` | Pure Vietnamese; English kept ONLY for proper nouns / product / technology names with no Vietnamese equivalent (code, Java, PostgreSQL, REST API...) | "Hệ thống chạy trên PostgreSQL" |
     | `vi-en` | Vietnamese with an English gloss in parentheses on FIRST use of each specialized term; register all glossed terms in `glossary.md` | "Cơ sở dữ liệu (database) lưu trữ hồ sơ" |

     The distinction between the last two: in `vi`, English appears because *no Vietnamese equivalent exists*; in `vi-en`, English glosses are used proactively to *teach terminology* so readers can research further.
   - **Fallback:** if either language setting is empty or missing, ASK the user to decide before running any command. Never assume a default.
3. **Write current state, not changes — no backward compatibility:** when creating or updating any block/document (new or adjustment), always treat it as the **first version**. Write the final content as if written from scratch today. A document describes how things ARE, never how they CHANGED. Do NOT keep backward compatibility: never mention backward, previous version, migration from old, or version history. Forbidden in document bodies: change-log phrasing such as "updated...", "added...", "removed...", "no longer applies...", "previously...", "backward compatible", "previous version", "migration". Do not embed version history, revision notes, or "what's new" sections anywhere — version control is handled by **git alone**.

   | Wrong (in body) | Right |
   |-----------------|-------|
   | "The payment feature has been added to the billing module." | "The billing module includes a payment feature..." |
   | "The legacy report section was removed in this version." | *(delete the section entirely, leave no trace)* |

4. **No duplication:** check the glossary before defining a new term.
5. **File naming:** kebab-case with numeric prefix indicating reading order — e.g. `01-problem-statement.md`.
6. **Images/diagrams:** store in `99-assets/`, reference via relative paths; no inline base64.
7. **Block lifecycle:** each block has status `draft → review → approved`, tracked in `.haro-docs/status/`; only `approved` blocks may be aggregated into deliverables without further review.
8. **Sub-READMEs:** every folder must have a `README.md` describing its scope and file list.

## 10. Command `/haro-docs review <topic|file>` — Critical Review via Subagent Reviewers

Objectively research, analyze and evaluate a problem, idea, or doc file using critical thinking and logic. The skill dispatches reviewer subagent(s) ("đệ tử"), then synthesizes their reports into one verdict plus follow-up proposals. **Read-only on docs**: never edits the reviewed file; only optionally saves a report under `.haro-docs/reviews/` after asking.

### Syntax

```
/haro-docs review Should we use microservices for this project?
/haro-docs review 04-architecture/02-components.md
/haro-docs review --no-agents The current pricing model has a flaw
```

### Workflow

1. **Parse target + load context (selective RAG)** — read `knowledge/_index.md` (if exists), load only rows whose domain/tags match the topic. If the target is a doc path: read that file + nearby files in the same folder + glossary. If it is a free-text problem: lightly scan repo + doc-root for relevant evidence. State what was loaded (`sources: ...`) so reviewers can cite it.
2. **Resolve reviewers from `.haro-docs/agents.yaml`:**
   - If the file is missing: use a single inline `critic` with the default prompt from `templates/agents.yaml` (single-critic mode).
   - If the file exists: show enabled agents (`id | role`) and let the user multi-select (picker tool when available, otherwise numbered list). Pre-select `default_reviewer`. Default mode is **single critic**; multi-agent runs only when the user selects 2+ agents or the topic explicitly needs research + critique.
   - `--no-agents` flag forces single inline critic, ignoring the config (useful for debugging).
   - If the runtime has no subagent mechanism (no Task tool): **inline fallback** — run each selected reviewer sequentially in the current context, clearly labeled `Reviewer <id> (inline fallback)`, then synthesize. Never fail just because subagents are unavailable.
3. **Dispatch reviewers** — each reviewer receives: the topic/file content, the loaded context summary, and its own `role + prompt` from `agents.yaml`. Require a structured return:
   - `findings:` bullet list (claim → evidence `file:line` or `knowledge/<file>`)
   - `counter-arguments:` strongest opposing views
   - `verdict:` agree | conditionally-agree | disagree + reasons
   - `confidence:` high/medium/low per finding
   - `open questions:` what evidence is still missing
   Run selected agents in parallel when the runtime supports it.
4. **Synthesize (skill, not a subagent)** — cross-check reports into:
   - `Consensus:` points all reviewers agree on
   - `Conflicts:` points they disagree on (with who-says-what)
   - `Missing evidence:` what would settle the conflicts
   - `Overall verdict:` agree | conditionally-agree | disagree + 3–5 line rationale
   - `Risks & alternatives:` short list
   Reply in `language.response`. Be objective: report disagreements honestly instead of hiding them.
5. **Save report (ask first)** — ask `Save review report to .haro-docs/reviews/? (save / skip)`. On `save`, write `.haro-docs/reviews/RR-YYYYMMDD-HHmmss-<slug>.md` with frontmatter (`topic, verdict, reviewers, date, sources`) + the synthesis + per-reviewer summaries. Never save without asking.
6. **Next-step popup** — after the synthesis, always propose follow-ups (picker when available): e.g. `generate <related file>`, `remember <new fact surfaced>`, `review again with more evidence`, `stop`. Wait for the pick; do NOT auto-run.

### Examples

```
/haro-docs review Should we use microservices for this project?
/haro-docs review 05-security/01-threat-model.md
/haro-docs review --no-agents Is the current SLA realistic?
```

## 11. Command `/haro-docs config` — Config Hub (agents | conventions | language)

Central hub for skill configuration. Three branches, no `doc-root` here (changing doc-root is a heavy migrate — keep it in `init`).

### 11.1 Hub (no args)

```
/haro-docs config               → hub picker (read-only until a branch is chosen)
/haro-docs config agents        → straight into §11.2
/haro-docs config conventions   → straight into §11.3
/haro-docs config language      → straight into §11.4
```

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

1. Show current `language.response` + `language.documentation` from `.haro-docs/project-profile.yaml` (with `vi` vs `vi-en` definitions from Section 9).
2. Offer change with `en | vi | vi-en` options as applicable; confirm before writing back to `project-profile.yaml`.
