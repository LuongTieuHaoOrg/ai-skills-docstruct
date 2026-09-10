---
name: docstruct
description: Manage project documentation structure using the Atomic Content Blocks model. Use when the user wants to initialize a documentation structure for a new project, organize/refactor existing documentation, aggregate complete documents (BRD, PRD, SAD, FSD...) from existing content blocks, or manage project knowledge memory via remember/knowledge. Run /docstruct with no args to see current structure and command help.
---

# Docstruct — Documentation Structure Skill

## 1. Overview

`docstruct` organizes project documentation as **Atomic Content Blocks**: every small section of documentation is a separate markdown file, written exactly once (Single Source of Truth), then flexibly assembled into complete documents (BRD, PRD, SAD, FSD...).

Key strengths:

- **Standardization:** every project has an identical documentation structure, making quality comparable.
- **Atomicity:** each block is an independent object — easy to assign, track status, and version.
- **Flexibility:** output documents are just different "Views" aggregated from the same source blocks via the **Aggregation Matrix** (Section 7).
- **Knowledge memory:** project facts (business, technical, team, conventions) are stored as small domain-scoped files under `.docstruct/knowledge/` and selectively loaded like RAG — no vector DB needed.

## 1.1 Knowledge as Selective RAG (no vector DB)

Because vector search is not available, knowledge is organized for **filename + index** retrieval:

- Each fact is stored in a file named by domain: `business-*.md`, `technical-*.md`, `team-*.md`, `common-*.md`, `security-*.md`, etc. The agent chooses the name that best matches the content so it can be found without loading all memory.
- `.docstruct/knowledge/_index.md` is the single lookup table: for each file, one row with `file | domain | one-line summary | tags`. Agents read **only `_index.md`** (small) to decide which files to load for the current task, then load only those files. Never load the entire `knowledge/` directory by default.
- Use `_index.md` before every `init` or `generate` turn that needs project context: read `_index.md`, pick rows whose domain/tags match the task (e.g. business for BRD, technical/architecture for SAD), load only those files. Knowledge overrides scanned defaults when they conflict.

## 2. Workspace `.docstruct/`

The skill stores all configuration and state in `.docstruct/` at the project root:

```
.docstruct/
├── project-profile.yaml   # Project profile: type, audience, doc-root, language settings
├── schema.yaml            # Approved folder tree + aggregation matrix
├── status/                # Per-block status: DRAFT | UPDATING | RELEASED
├── knowledge/             # Project knowledge memory — selective RAG (see Section 4)
│   ├── _index.md          # Lookup: file | domain | summary | tags
│   ├── business-*.md
│   ├── technical-*.md
│   ├── team-*.md
│   └── common-*.md
└── elicitation/           # Interim Q&A for generate (see Section 6)
    └── <sanitized-path>.md
```

> **Important:** `project-profile.yaml` records the **doc-root** — the documentation location chosen by the user during `init`. Every command (`generate`, `remember`, `knowledge`) must read this config before operating. Never guess the doc-root.

> **Knowledge RAG:** Before any `init` or `generate` turn that needs project context, read `.docstruct/knowledge/_index.md` (if it exists), then selectively load only the knowledge files whose domain/tags match the task. Use knowledge as ground truth when it conflicts with scanned defaults. See Section 4.

> **Elicitation & Status:** `generate` stores interim Q&A in `.docstruct/elicitation/` and marks each doc file `DRAFT | UPDATING | RELEASED` (frontmatter `status:` + `.docstruct/status/`). See Section 6.

> **Language settings:** `project-profile.yaml` also records the **reply language** (`language.response`) and the **documentation language** (`language.documentation`). These are the single source of truth for all communication and content decisions — see Section 9. If they are empty or missing, ask the user before running any command.

## 3. Command `/docstruct` (no args) — Help & Status Dashboard

When the user runs `/docstruct` with no arguments, or with arguments that do not match any configured command, do NOT execute a workflow. Instead show the help dashboard:

1. **Read state** — if `.docstruct/project-profile.yaml` and `.docstruct/schema.yaml` exist, read `docroot` and list the actual folder tree under doc-root (for each of `00-common` → `99-assets` show exists/missing and file count). If not initialized, show `Not initialized` and display the standard tree from Section 8 as preview.
2. **Check knowledge** — if `.docstruct/knowledge/_index.md` exists, show `Knowledge: N files` and the first 5 index rows; otherwise show `Knowledge: (empty)`.
3. **Show command summary:**

   | Command | When to use | Example |
   |---------|-------------|---------|
   | `/docstruct init <description>` | Initialize structure (12 folders) | `/docstruct init E-commerce Next.js + PostgreSQL` |
   | `/docstruct generate` | Build next doc in order (role-adaptive Q&A) | `/docstruct generate` |
   | `/docstruct generate <file>` | Focus on a specific file | `/docstruct generate 02-business/01-value-prop.md` |
   | `/docstruct remember <free text>` | Record knowledge (analyze → confirm → save) | `/docstruct remember STID is my company` |
   | `/docstruct knowledge` | List all knowledge files | `/docstruct knowledge` |

4. **Show Aggregation Matrix (compact)** — BRD/PRD/SAD/FSD source folders from Section 7.
5. **Do not create or modify any file.** If the first token is unknown (e.g. `/docstruct foo`), prefix the dashboard with `Unknown command 'foo'. Valid: init, generate, remember, knowledge.` and suggest the closest match. Also handle `help`, `--help`, `-h` as aliases for this dashboard. Matching is case-insensitive, trim whitespace.

## 4. Command `/docstruct remember / knowledge` — Project Knowledge Memory (Selective RAG)

Knowledge files are project facts the agent must remember and follow. They are stored as small domain-scoped files so they can be loaded selectively without reading all memory.

### Storage

```
.docstruct/knowledge/
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
/docstruct remember STID is my company, I am PM, members are Hao (me), Vu (Backend) and Dai (Frontend)
/docstruct knowledge
```

## 5. Command `/docstruct init <project description>`

Initialize the documentation structure for a project.

### Required workflow

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
   - `.docstruct/docs/` (contained within the skill workspace)
4. **Confirm the outline** — present the folder tree + specific file list; wait for user approval. If the project already has non-conforming documentation, propose a reorganization plan (migration mapping table) in this step.
5. **Initialize** — after approval:
   - Create `.docstruct/project-profile.yaml` and `.docstruct/schema.yaml` (from templates in the skill's `templates/` directory), including the chosen `language.response` and `language.documentation`
   - Create the folder tree per the outline, each folder gets a `README.md` describing its scope
   - Create a root overview README at the doc-root including the reading path

### Examples

```
/docstruct init E-commerce project with Next.js + PostgreSQL, team of 3 devs
/docstruct init
```

With no description: still scan the current project first, then start asking from step 2.

## 6. Command `/docstruct generate` / `generate <file>` — Build Docs in Order (Role-Adaptive)

Build documentation flexibly based on current state + your role. Two modes:

- `/docstruct generate` — propose 2–3 next files to build, based on reality, not rigid order.
- `/docstruct generate <file>` — focus on a specific file (e.g. `02-business/01-value-prop.md` or `10-deliverables/01-BRD.md`) to create or adjust it. If the file is `RELEASED`, warn and ask to update.

### Ordered index

Canonical order is `00-common → 01-overview → 02-business → 03-features → 04-architecture → 05-security → 06-implementation → 07-quality → 08-operations → 09-guides → 10-deliverables → 99-assets` as defined in `schema.yaml` and Section 8. Within each folder, files are ordered by numeric prefix. This order is the **reference**, not a rigid gate: use it to understand what is prerequisite for what, but do not block flexibly. If all files are `RELEASED`, reply `All done — every file is RELEASED.`.

### Elicitation storage

Interim Q&A is stored in `.docstruct/elicitation/<sanitized-path>.md` (e.g. `02-business-01-value-prop.md`) to avoid context overload. It is read on demand during `generate` and not loaded by default. After the target file is completed, ask the user whether to keep or delete the elicitation file.

### Status

Each doc file carries `status:` in frontmatter and a mirror in `.docstruct/status/<path>`:

- `DRAFT` — brand-new file, first version.
- `UPDATING` — has reference content and is being edited.
- `RELEASED` — completed, may be referenced by later files, not edited. If `generate <file>` targets a `RELEASED` file, warn `File is RELEASED` and ask `Update? (yes/no)`. On `yes`, set status to `UPDATING` before editing.

No backward compatibility: every creation or adjustment is treated as the **first version** per Section 9.

### Workflow

1. **Selective RAG** — read `knowledge/_index.md` (if exists), pick rows whose domain/tags match the target file (`business` for BRD/Proposal, `technical`+`architecture` for SAD/FSD, `team` for people), load only those knowledge files.
2. **Ask who you are & propose next file based on reality:**
   - Ask first: `Who are you in this project? What is your role? (customer / sales / BA / dev / PM) — what do you know best about?` Save answer to `knowledge/team-role.md` if not yet stored (or confirm `Still <role>?`), and also to `elicitation/<target>.md`.
   - Scan the current docs reality: which files are missing/empty (`DRAFT`), which are `UPDATING`, which are `RELEASED`, which `change-requests/CR-*.md` are new. Use the canonical order as reference to understand prerequisites (e.g. `01-overview` and `02-business` are prerequisites for `04-architecture`), but **do not enforce rigidly** — understand flexibly what is actually needed.
   - If `<file>` was given: treat it as the user's preference, but if it depends on an unfinished prerequisite (e.g. targeting `04-architecture` while `01-overview/03-goals.md` is still empty), explain why the prerequisite matters and ask `Do you want to continue with this file or switch to the prerequisite?`.
   - If no `<file>`: propose **2–3 candidates** for the next file, each with a one-line reason (e.g. `1. 01-overview/01-problem-statement.md — foundational purpose is still empty; 2. 02-business/01-value-prop.md — business value needed before architecture`). Let the user pick. If the user picks none, they can specify another file.
   - Once a target is chosen/confirmed, read its existing content if any (`UPDATING` case) and nearby files in the same folder + glossary for context. Note current status (`DRAFT` if new, `UPDATING` if exists).
3. **Role-adaptive elicitation (flexible, not rigid):**
   - Generate **~5 tailored questions** based on `role + file domain + current reality` (e.g. for `02-business/*` with customer → market/pain/SLA/metrics; for `04-architecture/*` with customer → flows/business rules only, with dev → endpoints/schemas/NFR/diagrams). **Never ask a customer about code**, but otherwise adapt to what is actually missing — do not apply a hard rule like "purpose before technical" to every case.
   - Append Q&A to the elicitation file.
   - Analyze: check which required sections of the target file (per schema) are still missing. If incomplete, ask another 3–5 follow-up questions, append to elicitation file, repeat until sufficient.
4. **Propose outline** — list sections for the target file, annotating source (which block/folder and which knowledge files). Wait for outline approval.
5. **Write** — create/update the target file in its canonical folder (kebab-case + numeric prefix, Section 9) using current-state style (Section 9, no backward). Update `status:` frontmatter to `DRAFT` or `UPDATING` as appropriate. If knowledge facts were introduced, also update `knowledge/` if needed. Record `related_paths` if any.
6. **Elicitation cleanup** — ask `Keep elicitation file for reference or delete? (keep / delete)`. Act accordingly.
7. **Confirm & mark status** — ask `Mark this file as? (DRAFT / UPDATING / RELEASED)`. Default is `RELEASED` if the user says the file is done. Update frontmatter and `.docstruct/status/<path>` accordingly. `RELEASED` files become referenceable by later `generate` runs.

### Examples

```
/docstruct generate
/docstruct generate 02-business/01-value-prop.md
/docstruct generate 10-deliverables/01-BRD.md
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
docs/                        # or .docstruct/docs/ depending on user-chosen doc-root
├── 00-common/               # 01-glossary.md, 02-abbreviations.md, 03-references.md, 04-conventions.md, 05-traceability.md (auto)
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

`init` creates all folders/files above; each `10-deliverables/*.md` is a placeholder with its own headings (e.g. BRD: Business Goals/Stakeholders/Requirements; SAD: Components/Data Flow/Security/Deployment) plus one-line descriptions and `Ref: ../01-overview/...` links to source blocks — not identical templates. `00-common/05-traceability.md` is auto-generated by `generate` from `status: RELEASED` + `knowledge/_index.md`; `07-roadmap.md` is simple Version|Goal|Target|Status (bugfix not on roadmap, tracked in `change-requests/` + `07-quality/05-quality-metrics.md`); `change-requests/CR-*.md` holds each customer request by timestamp.

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
7. **Block lifecycle:** each block has status `draft → review → approved`, tracked in `.docstruct/status/`; only `approved` blocks may be aggregated into deliverables without further review.
8. **Sub-READMEs:** every folder must have a `README.md` describing its scope and file list.
