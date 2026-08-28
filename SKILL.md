---
name: docstruct
description: Manage, validate and generate project documentation structure using the Atomic Content Blocks model. Use when the user wants to initialize a documentation structure for a new project, organize/refactor existing documentation, audit docs quality and completeness, aggregate complete documents (BRD, PRD, SAD, FSD...) from existing content blocks, or run discussion sessions. Run /docstruct with no args to see current structure and command help.
---

# Docstruct — Documentation Structure Skill

## 1. Overview

`docstruct` organizes project documentation as **Atomic Content Blocks**: every small section of documentation is a separate markdown file, written exactly once (Single Source of Truth), then flexibly assembled into complete documents (BRD, PRD, SAD, FSD...).

Key strengths:

- **Standardization:** every project has an identical documentation structure, making quality comparable.
- **Atomicity:** each block is an independent object — easy to assign, track status, and version.
- **Flexibility:** output documents are just different "Views" aggregated from the same source blocks via the **Aggregation Matrix** (Section 6).

## 2. Workspace `.docstruct/`

The skill stores all configuration and state in `.docstruct/` at the project root:

```
.docstruct/
├── project-profile.yaml   # Project profile: type, audience, doc-root, language settings
├── schema.yaml            # Approved folder tree + aggregation matrix
├── status/                # Per-block status: draft / review / approved
└── chats/                 # Discussion sessions (see Section 4)
    ├── _active            # Text file containing the active chat id, if any
    └── <id>.md            # One file per session, id = [a-z0-9]{6-8}
```

> **Important:** `project-profile.yaml` records the **doc-root** — the documentation location chosen by the user during `init`. Every command (`validate`, `generate`, `chat`) must read this config before operating. Never guess the doc-root.

> **Language settings:** `project-profile.yaml` also records the **reply language** (`language.response`) and the **documentation language** (`language.documentation`). These are the single source of truth for all communication and content decisions — see Section 10. If they are empty or missing, ask the user before running any command.

> **Implicit chat mode:** At the start of EVERY turn (whether the user message has a `/docstruct` prefix or not), check `.docstruct/chats/_active`. If it exists and `chats/<id>.md` exists, treat the user's message as continuation of that chat session: update the minutes file per Section 4 and prefix your reply with `[chat: <id>]`. This enables implicit continuation without requiring `/docstruct chat` prefix. `chat close` removes `_active` to exit this mode.

## 3. Command `/docstruct` (no args) — Help & Status Dashboard

When the user runs `/docstruct` with no arguments, or with arguments that do not match any configured command, do NOT execute a workflow. Instead show the help dashboard:

1. **Read state** — if `.docstruct/project-profile.yaml` and `.docstruct/schema.yaml` exist, read `docroot` and list the actual folder tree under doc-root (for each of `00-common` → `99-assets` show exists/missing and file count). If not initialized, show `Not initialized` and display the standard tree from Section 9 as preview.
2. **Check active chat** — if `.docstruct/chats/_active` exists, show `Active chat: <id> (topic)` and a one-line summary from `chats/<id>.md`.
3. **Show command summary:**

   | Command | When to use | Example |
   |---------|-------------|---------|
   | `/docstruct init <description>` | Initialize structure (12 folders) | `/docstruct init E-commerce Next.js + PostgreSQL` |
   | `/docstruct validate [scope]` | Audit Structure/Content/Terminology/Formatting | `/docstruct validate 04-architecture` |
   | `/docstruct generate <type> <description>` | Aggregate a full document (BRD/PRD/SAD...) | `/docstruct generate SAD customer review` |
   | `/docstruct chat new [topic]` | Start a discussion session (implicit mode) | `/docstruct chat new Data model` |
   | `/docstruct chat -<id>` | Resume session `<id>` (strip leading `-`) | `/docstruct chat -a1b2c3` |
   | `/docstruct chat list` | List all sessions | `/docstruct chat list` |
   | `/docstruct chat close` | Exit implicit chat mode | `/docstruct chat close` |
   | `/docstruct chat delete -<id>` | Delete session `<id>` permanently | `/docstruct chat delete -a1b2c3` |

4. **Show Aggregation Matrix (compact)** — BRD/PRD/SAD/FSD source folders from Section 8.
5. **Do not create or modify any file.** If the first token is unknown (e.g. `/docstruct foo`), prefix the dashboard with `Unknown command 'foo'. Valid: init, validate, generate, chat.` and suggest the closest match. Also handle `help`, `--help`, `-h` as aliases for this dashboard. Matching is case-insensitive, trim whitespace.

## 4. Command `/docstruct chat ...` — Discussion Sessions

Discussion sessions store meeting-like minutes (decisions, open questions, next steps), not full transcripts. Presence of `.docstruct/chats/_active` means implicit chat mode is on (see Section 2): every subsequent user turn — even without `/docstruct` prefix — is treated as continuation of that session until `chat close`.

### Storage

```
.docstruct/chats/
├── _active          # contains active id, e.g. "a1b2c3" — absent means no active session
└── <id>.md          # id = [a-z0-9]{6-8}, one file per session
```

`_active` is the single source for whether a session is active. No `status` field is used. Deleting a session removes its file permanently.

### Chat file schema (`chats/<id>.md`)

```yaml
---
id: abc123
topic: "Data model for 04-architecture"
created: YYYY-MM-DD
updated: YYYY-MM-DD
related_paths: [04-architecture/03-data-model.md]
---
# Decisions
# Open Questions
# Next Steps
# New Terms (to sync into 00-common/glossary.md)
```

Body sections are concise minutes. Each discussion turn auto-updates this file: read the previous minutes, merge new decisions/open questions/next steps, update `updated` and `related_paths`.

### Subcommands

| Subcommand | Behaviour |
|------------|-----------|
| `chat new [topic]` | Generate id `[a-z0-9]{6-8}` (if user typed `-abc`, strip `-` → `abc`). Create `chats/<id>.md` with `topic` (default to first user message if empty), write `_active = id`, reply `[chat: <id>] Created — all following messages will be recorded here. Use /docstruct chat close to exit.` |
| `chat -<id>` | Resume: id is the token without leading `-` (e.g. `-a1b2c3` → `a1b2c3`). If `chats/<id>.md` exists, write `_active=id` and return its minutes summary. Otherwise error + `chat list`. |
| `chat list` | List all `chats/*.md` (excluding `_active`) as table `id | topic | updated | decisions#`, highlight `_active` row. Read-only. |
| `chat close` | Remove `_active` file. Reply `[chat: <id>] Closed — implicit mode off. Minutes kept at chats/<id>.md.` Do NOT delete the minutes file. |
| `chat delete -<id>` | Delete `chats/<id>.md` permanently (strip `-`). If `_active == id`, also remove `_active`. |

Unknown chat subcommand → prefix help: `Unknown chat command 'X'. Valid: new, -<id>, list, close, delete -<id>.` then show this table. `help`/`--help`/`-h` also show it.

### Implicit continuation rule

While `_active` exists, at the start of EVERY turn (even plain chat without `/docstruct`), the agent must read `_active` and `chats/<id>.md`, treat the user's message as discussion input, update the minutes file, and prefix the reply with `[chat: <id>]`. `chat close` is the only way to exit.

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
   - **Documentation language** — the language of doc content: `en`, `vi`, or `vi-en` (definitions in Section 10)
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

## 6. Command `/docstruct validate [scope]`

Audit the structure and quality of existing documentation.

### Workflow

1. **Determine scope** — from the argument, or default to everything: Structure, Content, Terminology, Formatting.
2. **Read configuration** — `.docstruct/project-profile.yaml` (doc-root) and `.docstruct/schema.yaml` (standard structure).
3. **Check:**
   - *Structure:* standard folders/files intact? Sub-READMEs present?
   - *Content:* leftover `[PLACEHOLDER]`, `TODO`, empty sections? Main sections filled?
   - *Terminology:* new terms defined in glossary?
   - *Formatting:* links working? File names follow convention (kebab-case + numeric prefix)?
4. **Write the report** to `<doc-root>/issues/docstruct-review-YYYY-MM-DD.md` per the table below.

### Severity criteria

| Severity | Definition | Examples |
|----------|-----------|----------|
| **Critical** | Structural deviation that breaks the documentation system | Missing standard folder; broken link; file in wrong location |
| **Major** | Content gaps affecting readers | Leftover `[PLACEHOLDER]`/`TODO`; large empty sections; missing sub-README |
| **Minor** | Convention drift, does not hinder usage | Term not in glossary; wrong kebab-case; missing numeric prefix |

### Example

```
/docstruct validate
/docstruct validate Only check folder structure and links in 04-architecture
```

## 7. Command `/docstruct generate <document type> <description>`

Write or update a complete document (BRD, PRD, SAD, FSD, Guide...).

### Workflow

1. **Identify the document type** — consult the **Aggregation Matrix** (Section 8) to determine which folders' blocks are needed.
2. **Review available blocks** — read source blocks; list blocks that **already exist** (reuse verbatim, do NOT rewrite) and blocks that are **missing**.
3. **Propose an outline** — list major/minor sections, annotating each part's source (which block, which folder) and what needs writing from scratch.
4. **Wait for outline approval** before writing.
5. **Write** — assemble existing blocks + write missing ones into their original folders (preserving SSOT); replace outdated content entirely and apply the current-state rule from Section 10; then aggregate into a complete file under `10-deliverables/` (or equivalent per schema).

### Example

```
/docstruct generate SAD System architecture document for customer review
/docstruct generate PRD Add the payment feature to the current PRD
```

## 8. Aggregation Matrix

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

## 9. Standard structure (single)

Single structure for all projects:

```
docs/                        # or .docstruct/docs/ depending on user-chosen doc-root
├── 00-common/               # Glossary, Abbreviations, References
├── 01-overview/             # Problem Statement, Summary, Goals, Scope
├── 02-business/             # Value Prop, Market, Pricing, Risk/Legal, SLA, use-cases/
├── 03-features/             # Detailed feature blocks, one group per feature
├── 04-architecture/         # System Overview, Components, Data Flow, API, Data Model
├── 05-security/             # Threat Model, Defense, Compliance, Privacy
├── 06-implementation/       # Deployment, Config, Integration, Migration
├── 07-quality/              # Test Strategy, Test Plans, UAT
├── 08-operations/           # Runbook, Incident Response, DR/BCP
├── 09-guides/               # User Guide, Admin Guide, Training, FAQ
├── 10-deliverables/         # Aggregated files: BRD, PRD, SAD, FSD...
└── 99-assets/               # Images, diagrams, templates
```

### Distinguishing `03-features` from `02-business/use-cases/`

These two folders cause the most confusion — distinguish by audience and detail level:

| | `03-features` | `02-business/use-cases/` |
|---|---------------|-------------------------------|
| Content | User stories + acceptance criteria + technical references | End-to-end interaction flows: main path, all alternative/exception paths |
| Audience | Developers, testers | Customers, BAs, test case writers |
| Language | Technical (schema/endpoint references allowed) | Pure business language, no technical detail |
| SSOT rule | Stories cross-link to use cases for context | Use cases contain NO technical detail |

One use case is typically decomposed into multiple user stories; each story links back to its source use case instead of retelling the flow.

Folders not yet needed may stay with their `README.md` and a short `> Out of scope for this project` note — `validate` will not flag them as Critical.

## 10. Authoring Rules

1. **Single Source of Truth:** each piece of content is written once in one file only; aggregated documents only assemble content, never duplicate it.
2. **Language configuration** — read from `project-profile.yaml` (`language.response`, `language.documentation`):
   - **Conversation replies** (clarifying questions, validate reports, outline proposals...) use `language.response`.
   - **Documentation content** uses `language.documentation`, defined as:

     | Mode | Definition | Example |
     |------|------------|---------|
     | `en` | Pure English | — |
     | `vi` | Pure Vietnamese; English kept ONLY for proper nouns / product / technology names with no Vietnamese equivalent (code, Java, PostgreSQL, REST API...) | "Hệ thống chạy trên PostgreSQL" |
     | `vi-en` | Vietnamese with an English gloss in parentheses on FIRST use of each specialized term; register all glossed terms in `glossary.md` | "Cơ sở dữ liệu (database) lưu trữ hồ sơ" |

     The distinction between the last two: in `vi`, English appears because *no Vietnamese equivalent exists*; in `vi-en`, English glosses are used proactively to *teach terminology* so readers can research further.
   - **Fallback:** if either language setting is empty or missing, ASK the user to decide before running any command. Never assume a default.
3. **Write current state, not changes:** when creating or updating any block/document, always write the final content as if written from scratch today. A document describes how things ARE, never how they CHANGED. Forbidden in document bodies: change-log phrasing such as "updated...", "added...", "removed...", "no longer applies...", "previously...". Do not embed version history, revision notes, or "what's new" sections anywhere — version control is handled by **git alone**.

   | Wrong (in body) | Right |
   |-----------------|-------|
   | "The payment feature has been added to the billing module." | "The billing module includes a payment feature..." |
   | "The legacy report section was removed in this version." | *(delete the section entirely, leave no trace)* |

4. **No duplication:** check the glossary before defining a new term.
5. **File naming:** kebab-case with numeric prefix indicating reading order — e.g. `01-problem-statement.md`.
6. **Images/diagrams:** store in `99-assets/`, reference via relative paths; no inline base64.
7. **Block lifecycle:** each block has status `draft → review → approved`, tracked in `.docstruct/status/`; only `approved` blocks may be aggregated into deliverables without further review.
8. **Sub-READMEs:** every folder must have a `README.md` describing its scope and file list.
