---
name: docstruct
description: Manage, validate and generate project documentation structure using the Atomic Content Blocks model. Use when the user wants to initialize a documentation structure for a new project, organize/refactor existing documentation, audit docs quality and completeness, or aggregate complete documents (BRD, PRD, SAD, FSD...) from existing content blocks.
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
├── project-profile.yaml   # Project profile: type, audience, scale tier, doc-root, language settings
├── schema.yaml            # Approved folder tree + aggregation matrix
└── status/                # Per-block status: draft / review / approved
```

> **Important:** `project-profile.yaml` records the **doc-root** — the documentation location chosen by the user during `init`. Every command (`validate`, `generate`) must read this config before operating. Never guess the doc-root.

> **Language settings:** `project-profile.yaml` also records the **reply language** (`language.response`) and the **documentation language** (`language.documentation`). These are the single source of truth for all communication and content decisions — see Section 8. If they are empty or missing, ask the user before running any command.

## 3. Command `/docstruct init <project description>`

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
   - **Documentation language** — the language of doc content: `en`, `vi`, or `vi-en` (definitions in Section 8)
3. **Propose a scale tier** — based on scan data, propose one of three tiers with reasoning (see Section 7):
   - **S** (small project, 1-2 people, MVP): 5–7 core folders
   - **M** (medium project, multiple modules): 8–10 folders
   - **L** (full product, many stakeholders): all 11 standard folders + assets
4. **Ask for the doc-root** — the user chooses:
   - `docs/` (traditional documentation folder), or
   - `.docstruct/docs/` (contained within the skill workspace)
5. **Confirm the outline** — present the folder tree + specific file list; wait for user approval. If the project already has non-conforming documentation, propose a reorganization plan (migration mapping table) in this step.
6. **Initialize** — after approval:
   - Create `.docstruct/project-profile.yaml` and `.docstruct/schema.yaml` (from templates in the skill's `templates/` directory), including the chosen `language.response` and `language.documentation`
   - Create the folder tree per the outline, each folder gets a `README.md` describing its scope
   - Create a root overview README at the doc-root including the reading path

### Examples

```
/docstruct init E-commerce project with Next.js + PostgreSQL, team of 3 devs
/docstruct init
```

With no description: still scan the current project first, then start asking from step 2.

## 4. Command `/docstruct validate [scope]`

Audit the structure and quality of existing documentation.

### Workflow

1. **Determine scope** — from the argument, or default to everything: Structure, Content, Terminology, Formatting.
2. **Read configuration** — `.docstruct/project-profile.yaml` (doc-root, tier) and `.docstruct/schema.yaml` (standard structure).
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
/docstruct validate Only check folder structure and links in 02-architecture
```

## 5. Command `/docstruct generate <document type> <description>`

Write or update a complete document (BRD, PRD, SAD, FSD, Guide...).

### Workflow

1. **Identify the document type** — consult the **Aggregation Matrix** (Section 6) to determine which folders' blocks are needed.
2. **Review available blocks** — read source blocks; list blocks that **already exist** (reuse verbatim, do NOT rewrite) and blocks that are **missing**.
3. **Propose an outline** — list major/minor sections, annotating each part's source (which block, which folder) and what needs writing from scratch.
4. **Wait for outline approval** before writing.
5. **Write** — assemble existing blocks + write missing ones into their original folders (preserving SSOT); replace outdated content entirely and apply the current-state rule from Section 8; then aggregate into a complete file under `07-deliverables/` (or equivalent per schema).

### Example

```
/docstruct generate SAD System architecture document for customer review
/docstruct generate PRD Add the payment feature to the current PRD
```

## 6. Aggregation Matrix

Applies to tier L (adjust accordingly when tier S/M removes folders):

| Document | Assembled from |
|----------|----------------------------|
| **BRD** | 01-overview + 06-business-docs |
| **PRD** | 01-overview + 03-features + 06-business-docs + 08-quality-assurance |
| **SAD** | 02-architecture + 04-security + 05-implementation-guide + 08-quality-assurance + 09-operations |
| **FSD** | 03-features + 02-architecture |
| **SRD** | 04-security + 03-features |
| **Proposal** | 01-overview + 06-business-docs |
| **Test Plan** | 08-quality-assurance + 03-features + 04-security |
| **Runbook** | 09-operations + 05-implementation-guide + 02-architecture |
| **User/Admin Guide** | 10-enablement + 03-features + 01-overview |

## 7. Default structure (tier L)

When proposing tier L, use the full schema below. Tier S/M reduces it per the guidance below.

```
docs/                        # or .docstruct/docs/ depending on user-chosen doc-root
├── 00-common/               # Glossary, Abbreviations, References
├── 01-overview/             # Problem Statement, Summary, Goals, Scope
├── 02-architecture/         # System Overview, Components, Data Flow, API, Data Model
├── 03-features/             # Detailed feature blocks, one group per feature
├── 04-security/             # Threat Model, Defense, Compliance, Privacy
├── 05-implementation-guide/ # Deployment, Config, Integration, Migration, Operations
├── 06-business-docs/        # Value Prop, Market, Pricing, Risk/Legal, SLA, use-cases/
├── 07-deliverables/         # Aggregated files: BRD, PRD, SAD, FSD...
├── 08-quality-assurance/    # Test Strategy, Test Plans, UAT
├── 09-operations/           # Runbook, Incident Response, DR/BCP
├── 10-enablement/           # User Guide, Admin Guide, Training, FAQ
└── assets/                  # Images, diagrams, templates
```

### Distinguishing `03-features` from `06-business-docs/use-cases/`

These two folders cause the most confusion — distinguish by audience and detail level:

| | `03-features` | `06-business-docs/use-cases/` |
|---|---------------|-------------------------------|
| Content | User stories + acceptance criteria + technical references | End-to-end interaction flows: main path, all alternative/exception paths |
| Audience | Developers, testers | Customers, BAs, test case writers |
| Language | Technical (schema/endpoint references allowed) | Pure business language, no technical detail |
| SSOT rule | Stories cross-link to use cases for context | Use cases contain NO technical detail |

One use case is typically decomposed into multiple user stories; each story links back to its source use case instead of retelling the flow.

**Tier S/M reduction guidance** — keep the minimum core set, prioritized in order:

| Tier | Keep | Remove |
|------|------|--------|
| **S** (5–7) | 00-common, 01-overview, 02-architecture, 03-features, 07-deliverables (+ assets if needed) | The rest |
| **M** (8–10) | Tier S + 04-security + 05-implementation-guide (+ 08 or 09 as needed) | 06, 09, 10 |

When reducing, update the Aggregation Matrix accordingly (assemble blocks from remaining folders); documents requiring removed sections get an explicit "N/A — out of documentation scope" note instead of silent omission.

## 8. Authoring Rules

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
6. **Images/diagrams:** store in `assets/`, reference via relative paths; no inline base64.
7. **Block lifecycle:** each block has status `draft → review → approved`, tracked in `.docstruct/status/`; only `approved` blocks may be aggregated into deliverables without further review.
8. **Sub-READMEs:** every folder must have a `README.md` describing its scope and file list.
